# Supabase Keep-Alive Setup Guide

This document explains how to set up a GitHub Actions workflow that keeps your Supabase database active by preventing it from going to sleep due to inactivity. This guide can be used for any Supabase project.

## 📋 Overview

**What it does:**
- Automatically pings your Supabase database on a custom schedule (default: twice per week)
- Prevents Supabase free tier databases from pausing due to inactivity
- Can be manually triggered when needed
- Provides clear success/failure feedback in GitHub Actions logs

**Why it's needed:**
Supabase free tier databases automatically pause after a period of inactivity (typically 1 week). This workflow ensures your database stays active by performing regular, lightweight queries.

## 🛠 Prerequisites

Before setting up this workflow, ensure you have:

1. **GitHub Repository**: Your project must be hosted on GitHub
2. **Supabase Project**: An active Supabase project with database access
3. **Repository Permissions**: Admin access to configure GitHub repository secrets
4. **Database Table**: At least one accessible table in your database

## 📁 File Structure

The workflow file should be located at:
```
.github/workflows/keep-supabase-active.yml
```
(You can rename this file to match your project needs)

## ⚙️ Setup Instructions

### Step 1: Create the Workflow File

1. In your GitHub repository, create the directory structure:
   ```
   .github/workflows/
   ```

2. Create the file `keep-supabase-active.yml` in this directory

3. Add the workflow content (see example workflow below)

#### Example Workflow File:
```yaml
name: Keep Supabase Active

on:
  schedule:
    - cron: '0 9 * * 1,4'  # Monday & Thursday 9 AM UTC
  workflow_dispatch:

jobs:
  ping-database:
    runs-on: ubuntu-latest
    steps:
      - name: Install Supabase JS
        run: npm install @supabase/supabase-js
        
      - name: Ping Supabase Database
        env:
          SUPABASE_URL: ${{ secrets.SUPABASE_URL }}
          SUPABASE_KEY: ${{ secrets.SUPABASE_KEY }}
        run: |
          node -e "
          const { createClient } = require('@supabase/supabase-js');
          console.log('🚀 Starting Supabase ping...');
          const supabase = createClient(process.env.SUPABASE_URL, process.env.SUPABASE_KEY);
          supabase.from('your_table_name').select('id').limit(1)
            .then(result => {
              if (result.error) {
                console.log('❌ Failed:', result.error.message);
                process.exit(1);
              } else {
                console.log('✅ Success! Found', result.data?.length || 0, 'records');
              }
            })
            .catch(error => {
              console.log('❌ Error:', error.message);
              process.exit(1);
            });
          "
```

**Important**: Replace `your_table_name` with an actual table name from your database.

### Step 2: Configure GitHub Secrets

You need to add two secrets to your GitHub repository:

#### Adding Secrets:
1. Go to your GitHub repository
2. Click on **Settings** tab
3. In the sidebar, click **Secrets and variables** → **Actions**
4. Click **New repository secret**

#### Required Secrets:

**Secret 1: SUPABASE_URL**
- **Name**: `SUPABASE_URL`
- **Value**: Your Supabase project URL
- **How to find**: 
  - Go to your Supabase dashboard
  - Select your project
  - Go to Settings → API
  - Copy the "Project URL"
  - Example: `https://your-project-id.supabase.co`

**Secret 2: SUPABASE_KEY**
- **Name**: `SUPABASE_KEY`
- **Value**: Your Supabase anon/public key
- **How to find**:
  - Go to your Supabase dashboard
  - Select your project
  - Go to Settings → API
  - Copy the "anon public" key (starts with `eyJ...`)

### Step 3: Configure Database Table Access

Choose a table to query for the keep-alive ping. This should be:

1. **Existing table**: A table that exists in your database
2. **Accessible**: RLS policies allow reading from this table with your anon key
3. **Modify the workflow**: Update the table name in the workflow (line ~24):
   ```javascript
   supabase.from('your_table_name').select('id').limit(1)
   ```

**Common table options:**
- `users` or `profiles` (if you have user management)
- `posts`, `items`, or `products` (main content tables)  
- Any public table with at least one record

### Step 4: Test the Setup

#### Manual Test:
1. Go to your GitHub repository
2. Click **Actions** tab
3. Find your keep-alive workflow (by the name you gave it)
4. Click **Run workflow** → **Run workflow**
5. Monitor the execution and check logs

#### Check Logs:
- Successful run should show: `✅ Success! Found X records`
- Failed run will show: `❌ Failed:` or `❌ Error:` with details

## 📅 Schedule Details

**Default Schedule:**
- **Days**: Monday and Thursday  
- **Time**: 9:00 AM UTC
- **Cron Expression**: `'0 9 * * 1,4'`

**To modify the schedule:**
Edit line 5 in the workflow file:
```yaml
- cron: '0 9 * * 1,4'  # Your custom schedule
```

**Cron Schedule Examples:**
- Daily at 9 AM UTC: `'0 9 * * *'`
- Every 3 days at 9 AM UTC: `'0 9 */3 * *'`
- Twice weekly (Monday & Friday): `'0 9 * * 1,5'`

## 🔧 Customization Options

### Change Target Table
If you want to ping a different table, modify the query:
```javascript
supabase.from('your_table_name').select('id').limit(1)
```

### Add More Health Checks
You can extend the script to check multiple tables or perform additional health checks:
```javascript
// Example: Check multiple tables
const tables = ['users', 'posts', 'products'];
for (const table of tables) {
  const result = await supabase.from(table).select('id').limit(1);
  // Handle results...
}
```

### Modify Workflow Name
Change the name on line 1:
```yaml
name: Your Custom Workflow Name
```

## 🚨 Troubleshooting

### Common Issues:

**1. "❌ Failed: relation 'your_table_name' does not exist"**
- **Solution**: The table doesn't exist or has a different name
- **Fix**: Create the table or update the table name in the workflow

**2. "❌ Failed: permission denied for table your_table_name"**
- **Solution**: RLS policies are blocking access
- **Fix**: Update your RLS policies to allow reading from the table

**3. "❌ Error: Invalid API key"**
- **Solution**: The SUPABASE_KEY secret is incorrect
- **Fix**: Verify and update the secret with the correct anon key

**4. "❌ Error: Invalid URL"**
- **Solution**: The SUPABASE_URL secret is incorrect
- **Fix**: Verify and update the secret with the correct project URL

**5. Workflow not running automatically**
- **Solution**: Check if the repository is active and workflow is enabled
- **Fix**: Ensure the workflow file is in the main/master branch

### Debugging Steps:

1. **Check Secrets**: Verify both secrets are correctly set
2. **Test Manually**: Run the workflow manually to see immediate results
3. **Check Logs**: Review the detailed logs in GitHub Actions
4. **Verify Database**: Ensure your Supabase project is active and accessible
5. **Test Connection**: Try connecting to your database using the same credentials locally

## 📊 Monitoring

### Success Indicators:
- ✅ Green checkmark in GitHub Actions
- Log message: "✅ Success! Found X records"
- Supabase database remains active

### Failure Indicators:
- ❌ Red X in GitHub Actions
- Error messages in logs
- Database may pause due to inactivity

### Notifications:
GitHub will email you if workflows consistently fail. You can also:
- Set up custom notifications in GitHub Actions
- Monitor via GitHub mobile app
- Create additional alerting mechanisms

## 🔒 Security Considerations

1. **Use Anon Key**: Only use the public/anon key, never the service role key
2. **Minimal Permissions**: The query only reads one record, minimizing security risk
3. **Regular Updates**: Keep @supabase/supabase-js dependency updated
4. **Secret Management**: Regularly rotate your Supabase keys if needed

## 📈 Performance Impact

- **Database Load**: Minimal - only queries 1 record twice per week
- **GitHub Actions Usage**: Uses ~1-2 minutes of your monthly GitHub Actions quota
- **Network Traffic**: Negligible - single lightweight API call

## 🤝 Contributing

If you improve this workflow:
1. Test thoroughly in your own repository
2. Document any changes
3. Consider sharing improvements with the team

---

**Need Help?**
- Check Supabase documentation: https://supabase.com/docs
- GitHub Actions documentation: https://docs.github.com/en/actions
- Review workflow logs for specific error messages 