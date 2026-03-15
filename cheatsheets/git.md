---
layout: page
title: "Git Cheat Sheet"
permalink: /cheatsheets/git/
---

_Die wichtigsten Git-Befehle f&uuml;r den Alltag._

## Setup & Config

```bash
git config --global user.name "Max Muster"
git config --global user.email "max@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase true          # rebase statt merge bei pull
git config --global core.editor "vim"
git config --list                             # alle Einstellungen anzeigen
```

## Repository

```bash
git init                                      # neues Repo erstellen
git clone https://github.com/user/repo.git   # Repo klonen
git clone --depth 1 URL                       # nur letzter Commit (schneller)
git remote -v                                 # Remotes anzeigen
git remote add origin URL                     # Remote hinzuf&uuml;gen
git remote set-url origin NEW_URL             # Remote URL &auml;ndern
```

## Grundlegende Workflow-Befehle

```bash
git status                                    # &Auml;nderungen anzeigen
git add file.txt                              # Datei stagen
git add .                                     # alle &Auml;nderungen stagen
git add -p                                    # interaktiv stagen (Hunk f&uuml;r Hunk)

git commit -m "Fix login bug"                 # Commit
git commit -am "Quick fix"                    # add + commit (nur tracked files!)
git commit --amend                            # letzten Commit &auml;ndern
git commit --amend --no-edit                  # nur Dateien nachtr&auml;glich hinzuf&uuml;gen

git push                                      # zum Remote pushen
git push -u origin feature-branch             # neuen Branch pushen + Tracking
git pull                                      # fetch + merge/rebase
git fetch                                     # nur holen, nicht mergen
```

## Branches

```bash
git branch                                    # lokale Branches anzeigen
git branch -a                                 # alle (inkl. remote) anzeigen
git branch feature-x                          # neuen Branch erstellen
git checkout feature-x                        # Branch wechseln
git checkout -b feature-x                     # erstellen + wechseln
git switch feature-x                          # Branch wechseln (modern)
git switch -c feature-x                       # erstellen + wechseln (modern)

git branch -d feature-x                      # Branch l&ouml;schen (safe)
git branch -D feature-x                      # Branch l&ouml;schen (force)
git push origin --delete feature-x            # Remote Branch l&ouml;schen
git branch -m old-name new-name               # Branch umbenennen
```

## Mergen & Rebasen

```bash
# Merge
git merge feature-x                           # feature-x in aktuellen Branch mergen
git merge --no-ff feature-x                   # immer Merge-Commit erstellen
git merge --abort                              # Merge abbrechen

# Rebase
git rebase main                                # aktuellen Branch auf main rebasen
git rebase -i HEAD~3                           # letzte 3 Commits interaktiv bearbeiten
git rebase --abort                             # Rebase abbrechen
git rebase --continue                          # nach Konfliktl&ouml;sung fortfahren

# Cherry-Pick
git cherry-pick abc123                         # einzelnen Commit &uuml;bernehmen
git cherry-pick abc123..def456                 # Bereich von Commits
```

## Stash

```bash
git stash                                      # &Auml;nderungen zwischenspeichern
git stash -m "WIP: new feature"                # mit Nachricht
git stash -u                                   # inkl. untracked files
git stash list                                 # alle Stashes anzeigen
git stash pop                                  # letzten Stash anwenden + l&ouml;schen
git stash apply stash@{2}                      # bestimmten Stash anwenden
git stash drop stash@{0}                       # Stash l&ouml;schen
git stash clear                                # alle Stashes l&ouml;schen
```

## History & Diff

```bash
git log                                        # Commit-History
git log --oneline                              # kompakt
git log --oneline --graph --all                # als ASCII-Graph
git log --author="Max" --since="2 weeks ago"
git log -p file.txt                            # &Auml;nderungen pro Commit f&uuml;r Datei
git log -S "searchTerm"                        # Commits die Text hinzuf&uuml;gen/entfernen

git diff                                       # unstaged &Auml;nderungen
git diff --staged                              # staged &Auml;nderungen
git diff main..feature                         # Unterschiede zwischen Branches
git diff HEAD~3                                # letzte 3 Commits

git show abc123                                # Commit Details anzeigen
git blame file.txt                             # Wer hat welche Zeile ge&auml;ndert?
git shortlog -sn                               # Commits pro Autor
```

## R&uuml;ckg&auml;ngig machen

```bash
# Staged &Auml;nderungen unstagen
git restore --staged file.txt
git reset HEAD file.txt                        # &auml;ltere Syntax

# &Auml;nderungen in Datei verwerfen
git restore file.txt
git checkout -- file.txt                       # &auml;ltere Syntax

# Letzten Commit r&uuml;ckg&auml;ngig machen (Code bleibt staged)
git reset --soft HEAD~1

# Letzten Commit r&uuml;ckg&auml;ngig machen (Code bleibt unstaged)
git reset HEAD~1

# Letzten Commit komplett verwerfen (VORSICHT!)
git reset --hard HEAD~1

# Commit r&uuml;ckg&auml;ngig machen mit neuem Commit (sicher f&uuml;r shared Branches)
git revert abc123

# Einzelne Datei auf Stand eines Commits zur&uuml;cksetzen
git restore --source=abc123 file.txt
```

## Tags

```bash
git tag v1.0.0                                 # Lightweight Tag
git tag -a v1.0.0 -m "Release 1.0"            # Annotated Tag
git tag -a v1.0.0 abc123                       # Tag auf bestimmten Commit
git tag                                        # Tags anzeigen
git push origin v1.0.0                         # Tag pushen
git push origin --tags                         # alle Tags pushen
git tag -d v1.0.0                              # lokalen Tag l&ouml;schen
git push origin --delete v1.0.0                # Remote Tag l&ouml;schen
```

## Aufr&auml;umen

```bash
git clean -n                                   # zeigen was gel&ouml;scht w&uuml;rde
git clean -fd                                  # untracked Dateien + Ordner l&ouml;schen
git gc                                         # Garbage Collection
git prune                                      # unerreichbare Objekte l&ouml;schen
git remote prune origin                        # gel&ouml;schte Remote-Branches aufr&auml;umen
git fetch --prune                              # fetch + prune
```

## Worktrees

```bash
git worktree add ../hotfix main                # neuen Worktree erstellen
git worktree list                              # alle Worktrees anzeigen
git worktree remove ../hotfix                  # Worktree entfernen
# Mehrere Branches gleichzeitig ausgecheckt!
```

## Bisect (Bug suchen)

```bash
git bisect start
git bisect bad                                 # aktueller Commit ist kaputt
git bisect good v1.0.0                         # dieser Commit war OK
# Git checkt automatisch Commits aus — du testest:
git bisect good                                # dieser ist OK
git bisect bad                                 # dieser ist kaputt
# Git findet den schuldigen Commit!
git bisect reset                               # zur&uuml;ck zum Ausgangszustand
```

## .gitignore

```gitignore
# Dateien
*.log
*.env
.DS_Store

# Ordner
node_modules/
dist/
build/
.idea/
target/

# Ausnahme
!important.log

# Patterns
**/temp/          # temp/ in jedem Unterordner
doc/**/*.pdf      # alle PDFs unter doc/
```

## Tipps & Aliase

```bash
# N&uuml;tzliche Aliase
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.unstage "reset HEAD --"
git config --global alias.last "log -1 HEAD"
git config --global alias.amend "commit --amend --no-edit"

# Schnelle Tricks
git commit --fixup=abc123       # Fixup-Commit f&uuml;r sp&auml;teren Rebase
git rebase -i --autosquash      # Fixups automatisch einordnen
git log --all --oneline -- file # Datei in allen Branches suchen
git reflog                      # alle HEAD-Bewegungen (Rettungsanker!)
```

## Quick Reference

```
Aktion                    Befehl
───────────────────────── ──────────────────────────
Neuer Branch              git switch -c name
Branch mergen             git merge name
Rebase                    git rebase main
Stash                     git stash / git stash pop
Letzten Commit &auml;ndern     git commit --amend
Commit r&uuml;ckg&auml;ngig        git revert hash
Hard Reset                git reset --hard HEAD~1
Datei wiederherstellen    git restore file.txt
Interaktiv stagen         git add -p
History durchsuchen       git log -S "text"
Bug finden                git bisect start
Reflog (Rettung!)         git reflog
```
