# CI/CD Deployment Setup — Next.js + VPS + GitHub Actions

This is our simple, repeatable process for deploying a Next.js application from GitHub to the VPS automatically.

## Final Flow

```text
Developer
    ↓
GitHub
    ↓
main branch
    ↓
GitHub Actions
    ↓
SSH as deploy user
    ↓
VPS
    ↓
git pull
    ↓
npm ci
    ↓
npm run build
    ↓
PM2 restart
    ↓
Nginx
    ↓
Website
```

---

# Part 1 — VPS Setup

## Step 1 — Create deployment user

Do this as `root`.

```bash
useradd -m -s /bin/bash deploy
```

Check:

```bash
id deploy
```

Expected:

```text
uid=... deploy
gid=... deploy
```

> We use `deploy` for application deployment.
> We do NOT use `root` for normal application deployment.

---

## Step 2 — Give deploy ownership of the application

Example:

```bash
chown -R deploy:deploy /var/www/testctsl.in
```

For another application, replace the application directory.

Example:

```bash
chown -R deploy:deploy /var/www/my-app
```

Check:

```bash
ls -ld /var/www/my-app
```

Expected:

```text
deploy deploy
```

---

# Part 2 — Test the Application as deploy

## Step 3 — Switch to deploy

```bash
su - deploy
```

Go to the application:

```bash
cd /var/www/testctsl.in
```

Check Git:

```bash
git status
```

Check Node:

```bash
node -v
```

Check npm:

```bash
npm -v
```

Check PM2:

```bash
pm2 -v
```

---

## Step 4 — Test the production build

Inside the application:

```bash
npm ci
```

Then:

```bash
npm run build
```

The build must succeed before continuing.

---

# Part 3 — PM2

## Step 5 — Run the application using deploy

Do NOT run the application with root PM2.

Example:

```bash
pm2 start npm --name testctsl.in -- start -- -p 3001
```

Replace:

```text
testctsl.in
3001
```

with the correct application name and port.

Check:

```bash
pm2 list
```

Expected:

```text
name          user
testctsl.in   deploy
```

---

## Step 6 — Test the application

Example:

```bash
curl -I http://localhost:3001
```

A valid HTTP response such as `200`, `301`, or `307` means the application is responding.

---

## Step 7 — Save PM2

```bash
pm2 save
```

This saves the PM2 process list for the `deploy` user.

---

# Part 4 — SSH Key

## Step 8 — Create SSH key

This only needs to be done once if we reuse the same deployment user.

On Windows:

```powershell
ssh-keygen -t ed25519 -C "github-actions-deploy"
```

This creates:

```text
id_ed25519
id_ed25519.pub
```

### Important

```text
id_ed25519
    ↓
PRIVATE KEY
Keep secret.

id_ed25519.pub
    ↓
PUBLIC KEY
Can be installed on the VPS.
```

---

# Part 5 — Allow GitHub Actions to SSH

## Step 9 — Create SSH directory

As `root`:

```bash
mkdir -p /home/deploy/.ssh
```

---

## Step 10 — Add the public key

Create:

```text
/home/deploy/.ssh/authorized_keys
```

Put the contents of:

```text
id_ed25519.pub
```

inside it.

Example:

```text
ssh-ed25519 AAAA... github-actions-deploy
```

---

## Step 11 — Set permissions

```bash
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
```

---

# Part 6 — Test SSH

## Step 12 — Test from Windows

```powershell
ssh -i C:\Users\asus\.ssh\id_ed25519 deploy@SERVER_IP
```

Example:

```powershell
ssh -i C:\Users\asus\.ssh\id_ed25519 deploy@147.93.28.163
```

It should log in **without asking for the deploy password**.

If successful:

```bash
exit
```

---

# Part 7 — GitHub Secrets

## Step 13 — Add GitHub Actions secrets

Go to:

```text
GitHub
→ Repository
→ Settings
→ Secrets and variables
→ Actions
```

Create:

```text
SERVER_HOST
SERVER_USER
SERVER_SSH_KEY
```

Values:

```text
SERVER_HOST = VPS IP
SERVER_USER = deploy
SERVER_SSH_KEY = private id_ed25519 key
```

### Important

Never put the private SSH key directly into the GitHub workflow.

Use:

```text
${{ secrets.SERVER_SSH_KEY }}
```

---

# Part 8 — GitHub Actions

## Step 14 — Create workflow

Create:

```text
.github/workflows/deploy.yml
```

Example:

```yaml
name: Deploy testctsl.in

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1.2.2
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /var/www/testctsl.in
            git pull origin main
            npm ci
            npm run build
            pm2 restart testctsl.in
```

For another application, change:

```text
/var/www/testctsl.in
```

to that application's directory.

And change:

```text
pm2 restart testctsl.in
```

to that application's PM2 name.

---

# Part 9 — Commit and Push

## Step 15 — Add workflow

From your local project:

```bash
git add .github/workflows/deploy.yml
```

Check:

```bash
git status
```

Then:

```bash
git commit -m "Add CI/CD deployment"
```

Push:

```bash
git push origin main
```

---

# Part 10 — Test CI/CD

## Step 16 — Check GitHub Actions

Go to:

```text
GitHub
→ Actions
```

Open the deployment workflow.

Expected:

```text
Deploy to VPS
    ↓
SSH
    ↓
git pull
    ↓
npm ci
    ↓
npm run build
    ↓
pm2 restart
    ↓
Success ✅
```

---

# Part 11 — Verify on VPS

SSH to the VPS:

```bash
ssh deploy@SERVER_IP
```

Go to the application:

```bash
cd /var/www/testctsl.in
```

Check PM2:

```bash
pm2 list
```

The application should show:

```text
user = deploy
status = online
```

---

# Important Rules

## Rule 1 — Root is for server administration

Use `root` for:

```text
Nginx
Firewall
Users
System packages
Server configuration
System services
```

---

## Rule 2 — deploy is for application deployment

Use `deploy` for:

```text
Git
npm
build
PM2
application files
```

---

## Rule 3 — GitHub Actions must NOT use root

Correct:

```text
GitHub Actions
      ↓
deploy
      ↓
application
```

Wrong:

```text
GitHub Actions
      ↓
root
      ↓
application
```

---

# Our Current testctsl.in Setup

```text
GitHub
   ↓
main
   ↓
GitHub Actions
   ↓
SSH
   ↓
deploy
   ↓
/var/www/testctsl.in
   ↓
npm ci
   ↓
npm run build
   ↓
PM2
   ↓
3001
   ↓
Nginx
   ↓
testctsl.in
```

---

# Repeat This for the Next Application

For each application, we mainly need to repeat:

```text
1. Application ownership → deploy
2. Test Git as deploy
3. npm ci
4. npm run build
5. Start PM2 as deploy
6. Assign/check its port
7. Test localhost
8. Create GitHub Actions workflow
9. Push to main
10. Test automatic deployment
```

The SSH key and GitHub secrets can be reused when using the same `deploy` user and VPS.

---

# Current Applications

We have identified these applications:

```text
testctsl.in
    → 3001

off-contract-nextjs
    → 3002

odisha-bizz-nextjs
    → 3000

meta-connect
    → 3004

nexsphere
    → 3005
```

We should migrate them **one application at a time**.

Do not change all applications simultaneously.

---

# Golden Rule

```text
LOCAL DEVELOPMENT
        ↓
GitHub
        ↓
Pull Request / main
        ↓
GitHub Actions
        ↓
deploy user
        ↓
PM2
        ↓
Nginx
        ↓
PRODUCTION
```

**No manual root deployment.**
