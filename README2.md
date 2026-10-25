# Site guide: cleanup, publishing and maintenance

Guide for the site at https://pavanaravapalli.github.io (Jekyll on GitHub Pages).

## 1. Clean up after running locally

Stop the local server first (Ctrl+C in the terminal). Then, inside the site folder:

```
rm -rf _site .jekyll-cache .sass-cache .bundle vendor Gemfile.lock
echo ".bundle/" >> .gitignore
```

These are build leftovers. GitHub rebuilds the site itself, so they never need to be pushed. Do not delete `Gemfile`, `.gitignore` or `_config.yml`.

Optional: if `brew install ruby` installed Ruby 4.0, `brew uninstall ruby` removes it. Brew warns you if anything else depends on it. Keep `ruby@3.3`, which is needed for local previews.

## 2. Publish to GitHub Pages

### One-time setup

1. **Hide your email in commits.** On GitHub go to Settings > Emails, tick "Keep my email addresses private", and copy the `...@users.noreply.github.com` address. Then run:

   ```
   git config --global user.name "Pavan Aravapalli"
   git config --global user.email "<that noreply address>"
   ```

   Without this, your real email is stored in the public history.

2. **Create the repository.** Go to https://github.com/new and name it exactly `pavanaravapalli.github.io`. Set it to **Public**, and leave "Add a README", ".gitignore" and "license" unticked.

3. **Push the files.** In the site folder run:

   ```
   git init
   git add .
   git status
   git commit -m "Initial site"
   git branch -M main
   git remote add origin https://github.com/pavanaravapalli/pavanaravapalli.github.io.git
   git push -u origin main
   ```

   Read the `git status` output before committing. It must not list `_site`, `vendor` or `Gemfile.lock`.

4. **Sign in.** GitHub no longer accepts your account password for pushes. Easiest option: `brew install gh`, then run `gh auth login` before the push. Alternative: create a Personal Access Token (Settings > Developer settings) with the `repo` permission and paste it when asked for a password.

5. **Turn on Pages.** In the repository go to Settings > Pages. Under "Build and deployment" set Source to **Deploy from a branch**, Branch **main**, Folder **/ (root)**, then Save.

6. **Wait 1 to 3 minutes.** The Actions tab shows a "pages build and deployment" run. When it has a green tick, open https://pavanaravapalli.github.io and check:
    - Home and About pages open
    - The topic filter buttons work
    - The tab icon and title look right
    - The Contact link opens your LinkedIn profile

### If something goes wrong

| Symptom