name: Deploy to Production (S3 + Slack + Rollback)

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Pour rollback

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install & Build
        run: |
          npm ci
          npm run build

      # === BACKUP AUTO VERS S3 (avant deploy) ===
      - name: Backup current site to S3
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: eu-west-3
        run: |
          BACKUP_DIR="backup-$(date +%Y%m%d-%H%M%S)-${{ github.sha }}"
          mkdir -p ./backup-tmp
          
          # Télécharge le site actuel depuis FTP
          lftp -u ${{ secrets.FTP_USER }},${{ secrets.FTP_PASS }} ${{ secrets.FTP_HOST }} <<EOF
          mirror /var/www/html/ ./backup-tmp/
          exit
          EOF
          
          # Upload backup vers S3
          aws s3 cp ./backup-tmp s3://${{ secrets.S3_BUCKET }}/backups/$BACKUP_DIR/ --recursive
          echo "Backup saved to s3://${{ secrets.S3_BUCKET }}/backups/$BACKUP_DIR/"

      # === DEPLOY VIA FTP ===
      - name: Deploy to FTP
        uses: SamKirkland/FTP-Deploy-Action@v4.3.5
        with:
          server: ${{ secrets.FTP_HOST }}
          username: ${{ secrets.FTP_USER }}
          password: ${{ secrets.FTP_PASS }}
          local-dir: ./dist/
          server-dir: /var/www/html/
          dangerous-clean-slate: true

      # === SLACK NOTIF SUCCESS ===
      - name: Slack Success
        if: success()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "Deploy PROD REUSSI !\n*Repo:* ${{ github.repository }}\n*Branch:* `${{ github.ref_name }}`\n*Commit:* <${{ github.event.head_commit.url }}|${{ github.sha | substr(0,7) }}>\n*Par:* @${{ github.actor }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

      # === SLACK + ROLLBACK ON FAILURE ===
      - name: Slack + Auto-Rollback on Failure
        if: failure()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "DEPLOY ECHOUE !\n*Repo:* ${{ github.repository }}\n*Commit:* `${{ github.sha | substr(0,7) }}`\n*Rollback auto lancé...*"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

      - name: Auto-Rollback from S3
        if: failure()
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: eu-west-3
        run: |
          # Récupère le dernier backup
          LATEST_BACKUP=$(aws s3 ls s3://${{ secrets.S3_BUCKET }}/backups/ | tail -1 | awk '{print $2}')
          echo "Rollback depuis : $LATEST_BACKUP"
          
          # Télécharge backup
          aws s3 cp s3://${{ secrets.S3_BUCKET }}/backups/$LATEST_BACKUP ./rollback-tmp/ --recursive
          
          # Redeploy via FTP
          lftp -u ${{ secrets.FTP_USER }},${{ secrets.FTP_PASS }} ${{ secrets.FTP_HOST }} <<EOF
          mirror -R --delete ./rollback-tmp/ /var/www/html/
          exit
          EOF
          
          echo "Rollback terminé. Site restauré."
