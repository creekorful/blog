+++
title = "wks - manage your codebase like a boss!"
date = "2020-02-22"
author = "Aloïs Micard"
authorTwitter = "" #do not include @
cover = ""
tags = ["My Projects", "Golang", "CLI"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

Don't you have many Git repositories laying everywhere on your disk? Isn't it a mess?

Well I have this problem too. From the forks I made to contribute, my personal projects, my work projects, etc... I have many repositories in my computer, sometimes with special settings (i.e using work email in work related repositories, signing commit with my GPG key in important repository, etc...)

And when I move to a new computer, I need to redo everything, again. This is really error prone.

Let me introduce you, wks (**W**or**KS**pace) the tool that will change that bad habit, and increase your productivity.

# How to use it?

## Create a workspace

The first thing to do is to create a workspace. A workspace is a collection of multiple git repositories, that may be organised into many sub-folders.

To get started you can either create a new workspace, or clone a remote workspace.

### Create a new workspace

To create a new workspace in the current directory, you'll just need to issue the following command:

```
wks init [<repo>]
```

where repo is the url of the upstream repository that will be used to save changes. If you don't want to save your changes remotely, you can omit this parameter.

### Clone a remote workspace

If you already have a remote workspace, you can clone it into the current directory by issuing the following command:

```
wks clone https://github.com/my-username/my-repo
```

## Add repositories

Once you have a workspace, you can add repositories into it by issuing the following command:

```
wks add <repo> [<dir>]
```

where repo is the url of the repository to add. You can optionally pass a directory to save the repository into.

The changes won't be committed to remote unless you run ``wks push``

### Examples

#### Add into workspace root

```
wks add https://github.com/creekorful/goreportcard-action
```

Will save the repository in the workspace with following path: ./goreportcard-action

#### Add into specific subdirectory

```
wks add https://github.com/creekorful/goreportcard-action Work
```

Will save the repository in the workspace with following path: ./Work/goreportcard-action

By using directories, you can group Git repositories by their context. For example you can create a Work directory for work related stuff, a Contributing directory for your open source contribution, a Personal directory for your personal project, and so on...

#### Override repository configuration

```
wks add --branch develop --config user.email test@example.com https://github.com/creekorful/goreportcard-action Github
```

You can also override repository configuration.

In this example develop will be cloned instead of master.
User email will be override by test@example.com. When you will clone the workspace on another computer, these changes will be applied automatically.

Issue ``wks help add`` for more details.

## Push config to remote

To save the current configuration in the remote, issue the following command:

```
wks push
```

## Pull config from remote

To retrieve change from remote, issue the following command:

```
wks pull
```

## Remove a repository

To remove repository from workspace, issue the following command:

```
wks rm <path>
```

Where path is the relative path to the repository to remove.

The changes won't be committed to remote unless you run wks push

### Examples

#### Remove repository from manifest

```
wks rm Team/test-repository
```

Remove test-repository located under the Team directory from workspace. This will not delete any files.

#### Remove repository from manifest & disk

```
wks rm Team/test-repository --delete
```

Remove test-repository located under the Team directory from workspace. This will also delete the repository from disk.

#### Update repositories

To update all repositories in workspace to their latest version, issue the following command:

```
wks update
```

## View workspace repositories

To list the repositories in workspace issue the following command:

```
wks ls
```

# Conclusion

The source code of wks is available on [Github](https://github.com/creekorful/wks). Please note that wks is currently under heavy development. Do no expect stable features before 1.0.0 is reached.

Pull requests & issues are welcomed!

Happy hacking!