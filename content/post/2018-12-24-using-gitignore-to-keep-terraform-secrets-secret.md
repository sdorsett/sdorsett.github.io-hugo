+++
title = "Using .gitignore to keep terraform secrets secret"
description = ""
tags = [
    "terraform",
    "git",
]
date = "2018-12-24T01:00:00Z"
categories = [
    "terraform",
    "git",
]
topics = [
    "learning-terraform-by-deploying-to-vsphere"
]
highlight = "true"
+++
This is the third in a series of posts that will walk you through using terraform to deploy and configure virtual machines on vsphere. In this post you will see how to use a .gitignore file to prevent a terraform.tfvar file from getting committed into a git repository. 

There is nothing vsphere specific in this post, but it is more about showing a pattern for keeping deployment specifics out of terraform code. This post assumes you have created a github account and are currently logged into that account.

---

### 1. create a new github repository for storing the terraform core that was created <a href="../2018-12-24-using-terraform-to-clone-a-virtual-machine-on-vsphere">in the previous blog post</a>.

Go to your github page, click the `+` in the upper right and select `new repository`:

![screenshot](/static/12242018-01-github-new-repository.png)

Name the repository and provide a description.

![screenshot](/static/12242018-02-new-repository-details.png)


### 3. Go the to directory that was created <a href="../2018-12-24-using-terraform-to-clone-a-virtual-machine-on-vsphere">in the previous blog post</a>. 

{{< highlight bash >}}
[root@terraform ~]# cd terraform-vsphere-clone/
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

### 4. Run `git init` to initialize the directory as a git repository

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# git init
Initialized empty Git repository in /root/terraform-vsphere-clone/.git/
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

### 5. Add the github repository as a remote using `git remote`

We need to let the git repository we just initialized know how to find the remote github repository we create. `git remote add origin <remote_git_url>' will add git remote to the local git repository. 

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# git remote add origin https://github.com/sdorsett/terraform-vsphere-clone.git
[root@terraform terraform-vsphere-clone]# git fetch --all
Fetching origin
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), done.
From https://github.com/sdorsett/terraform-vsphere-clone
 * [new branch]      master     -> origin/master
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

Running `git fetch --all` cause git to query the remote repository and pull down the remote state.

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# git pull origin master
From https://github.com/sdorsett/terraform-vsphere-clone
 * branch            master     -> FETCH_HEAD
Already up-to-date.
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

Running `git pull origin master` will ensure the latest versions of remote files are copied to the local repository.

### 6. Run `git status` to view the local files that have not been commited to the github repository

Running `git status` will now show the local files that do not exist in the remote github repository:

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# git status
# On branch master
#
# Initial commit
#
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#
#    .terraform/
#    main.tf
#    output.tf
#    terraform.tfstate
#    terraform.tfstate.backup
#    terraform.tfvars
#    variables.tf
nothing added to commit but untracked files present (use "git add" to track)
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

In the above output there are several directories and files we have not talked about before:

* .terraform/ - a hidden directory where terraform stores the providers pulled down when running `terraform init`

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# tree .terraform
.terraform
└── plugins
    └── linux_amd64
        ├── lock.json
        └── terraform-provider-vsphere_v1.9.0_x4

2 directories, 2 files
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

* terraform.tfstate - the file terraform uses to store the state after the last run of `terraform apply` 
* terraform.tfstate.backup - terraform will backup `terraform.tfstate` to `terraform.tfstate.backup` in order to preserve the state of the previous run of `terraform apply`

All of these files should not be added into a remote git repository since:

* .terraform/ - this directory will be re-created when `terraform init` is run in a new environment.
* terraform.tfstate and terraform.tfstate.backup - these files contain the terraform state specific of a specific environment and do not need to be preserved in a repository.
* terraform.tfvars - contain secrets (usernames, password, ip addresses, etc) about a specific environment.

### 7. Create a `.gitignore` file to prevent these files from being added to the remote repository 

Create a `.gitignore` file in the same directory and add the following files

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# vim .gitignore
[root@terraform terraform-vsphere-clone]# cat .gitignore
.terraform/
terraform.tfstate
terraform.tfstate.backup
terraform.tfvars
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

Re-running `git status` will now show all the local files that have not been committed to the remote repository.

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# git status
# On branch master
#
# Initial commit
#
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#
#    .gitignore
#    main.tf
#    output.tf
#    variables.tf
nothing added to commit but untracked files present (use "git add" to track)
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

You can see that `.gitignore` is being displayed, but the file names we added to the `.gitinore` file are ignored.

### 8. Add the uncommited local files and commit the changes

Running `git add .` will stage all modified files to be added to a commit:

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# git add .
[root@terraform terraform-vsphere-clone]# git status
# On branch master
#
# Initial commit
#
# Changes to be committed:
#   (use "git rm --cached <file>..." to unstage)
#
#    new file:   .gitignore
#    new file:   main.tf
#    new file:   output.tf
#    new file:   variables.tf
#
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

`git commit -m "<commit_message>"` will commit the staged files.

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# git commit -m "added main.tf, output.tf and variables.tf files"
[master cc5c928] added main.tf, output.tf and variables.tf files
 4 files changed, 97 insertions(+)
 create mode 100644 .gitignore
 create mode 100644 main.tf
 create mode 100644 output.tf
 create mode 100644 variables.tf
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

### 9. Push the committed changes to the remote github repository

`git push origin master` will push the local changes to the remote github repository

{{< highlight bash >}}
[root@terraform terraform-vsphere-clone]# git push origin master
Username for 'https://github.com': sdorsett
Password for 'https://sdorsett@github.com':
Counting objects: 7, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (6/6), done.
Writing objects: 100% (6/6), 1.23 KiB | 0 bytes/s, done.
Total 6 (delta 0), reused 0 (delta 0)
To https://github.com/sdorsett/terraform-vsphere-clone.git
   70e13d6..cc5c928  master -> master
[root@terraform terraform-vsphere-clone]#
{{< /highlight >}}

#### I hope this post has been useful in helping explain how to use environment variables to override terraform variables. This pattern will be used in future posts to keep the details of how to connect to vsphere from being defined in terraform code.

### 10. Checking the github remote repository to validate the files were committed.

We can now refresh the github remote repository and validate the files were committed 

![screenshot](/static/12242018-03-changes-pushed-to-github-repository.png)

The files are present in the repository and the latest commit message matches the local commit that was made.

#### Hopefully you found this post helpful in understanding how to use a .gitignore file to keep environment specific secrets out of a public github repository. The repository that was used in this post can be <a href="https://github.com/sdorsett/terraform-vsphere-clone">found here</a>.
 

---

#### Please provide any feedback or suggestions to my twitter account located on the about page.
