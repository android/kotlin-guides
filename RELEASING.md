Releasing
=========

 1. Checkout `master` (`git checkout master`)
 2. Pull latest changes from upstream (`git pull`)
 3. Create an entry in `_changes/` for the current date and write in the changes since the last version. Easiest way to do this is to copy the previous file and update it. (Yes those double `---` lines at the top of each file are required)
 4. Commit (`git commit -am "Prepare release 20XX-YY-ZZ"`)
 5. Tag (`git tag -a 20XX-YY-ZZ -m "20XX-YY-ZZ"`)
 6. Checkout `gh-pages` branch (`git checkout gh-pages`)
    * If you don't have a `gh-pages` branch locally, run `git checkout -t origin/gh-pages`.
 7. Merge in `master` (`git merge master`)
 8. Push branches (`git push`)
 9. Push tag (`git push --tags`)
