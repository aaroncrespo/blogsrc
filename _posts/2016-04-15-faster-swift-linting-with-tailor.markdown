---
layout: post
title: "Faster Swift Linting with Tailor"
date: 2016-04-15 10:30
comments: true
categories: tools, code quality
---

Linting is a good practice that helps make code reviews smoother.  
On linted code we can focus reviews on what the code is actually doing over what the code looks like. 
It also acts as a hedge against bike-shedding in a project and we can focus the review on the big ideas in the change. 
If we ever need discussion around code formatting some people have a strong "tabs vs spaces" opinion. 
Our style guide and lint rules are their own git repository a these discussions happen in there separate from the code that is subject to it. 

We use `tailor.sh` to run these quick formatting checks against our Swift style guide.
In a recent project we noticed lint processing times were growing at a steady enough pace, to the point where sometimes we would turn off and forget to turn it back on.
We looked at how `tailor.sh` worked and noticed it would lint files that passed earlier checks but were not modified. 
We looked for flags to pass in to change this behavior, but ended up solving it on the input side.

The default tailor integration is a small bash script executed as part of build phase:  

```bash
if hash tailor 2>/dev/null; then
    tailor
else
    echo "warning: Please install Tailor from https://tailor.sh"
fi
```

We updated this to trim input into tailor and become more efficient with our time while working on a large swift project.

```bash
if hash tailor 2>/dev/null; then
    #only run non deleted dirty files through tailor
    FILES=`(git ls-files -mo --exclude-standard; git diff origin/HEAD --diff-filter=ACMRUXB --name-only) 
            | grep swift`

    if [ -n "$FILES" ]; then
        echo $FILES | xargs tailor
    fi
else
    echo "warning: Please install Tailor from https://tailor.sh"
fi
```

# Breaking it down 

1. The `if hash .. then .. fi` is gracefully handling tailor not being installed.
1. Next We are going to store all our dirty or new files in a local Bash variable `FILES`
1. Next is where it differs from the default 
    1. `git ls-files -mo --exclude-standard` 
        A flat list of files, showing modified `-m` and untracked `-o` respecting your `--exclude-standard` .gitignore. 
    1. `git diff origin/HEAD --diff-filter=ACMRUXB --name-only`
        A diff of the local content against the `HEAD` of your default remote repository. Showing only the file names of any different files except deleted.  A `D` option in the diff-filter option would show deleted files and is a default)
1. `() |` Combine the output of the first two git commands and send that into
1. `grep swift` We only want to run Swift files into tailor. So filter any file that isn't Swift source code. 
1. The `xargs` step just allows us to skip tailor all together if there is nothing to tailor.

# Notes
The reason we use two separate git commands is that we like to see tailor results against our remote even if all local changes are added to the local index. Files aren't listed in `git ls-files` with our options once added to the index.
We use `git ls-files` over `git status` because we don't particularly care about the status of a file which git status provides per file. Rather we want a list of files matching our criteria.

#Finally
Recently `tailor.sh` added parallel processing so running this task happens on multiple files at once, earlier versions of tailor processed one file at a time and when run as a compile step iteration times where much higher. 
Running 4 times as fast can still eat up surprising amounts of time when you are trying to run tight iteration cycles. 