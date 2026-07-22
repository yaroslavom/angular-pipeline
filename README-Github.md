# Hands-on CodePipeline-01 : Creating a CI/CD Pipeline via AWS CodePipeline with GitHub

Purpose of this hands-on training is to create a CI/CD pipeline for an Angular application using **GitHub** as the source repository and **AWS CodePipeline** for continuous integration and deployment.

---

## Learning Outcomes

At the end of this hands-on training, students will be able to:

- Understand the concepts of CI/CD.
- Create a CI/CD pipeline using AWS CodePipeline.
- Create an S3 bucket for static website deployment.
- Use **GitHub** as the version control system.
- Use AWS CodeBuild to build an Angular project.
- Use a `buildspec.yml` file.
- Integrate automated testing into CodePipeline using CodeBuild.

---

## Outline

- Part 1 - Creating an S3 Bucket for Deployment
- Part 2 - Creating a GitHub Repository and Connecting It to the Local Machine
- Part 3 - Creating the CodePipeline
- Part 4 - Integrating Testing into CodePipeline

---

# PART 1 - Creating a Bucket and Route 53 Record for Deployment

## Section 1: Create the S3 Bucket

Create a bucket with the following properties:

```text
Bucket name                 : myangular.<domainname>
Region                      : N. Virginia
Object Ownership            : ACLs enabled - Bucket owner preferred
Block all public access     : Unchecked
Versioning                  : Disabled
Tagging                     : 0 Tags
Object-level logging        : Disabled
```

After creating the bucket, enable **Static Website Hosting**.

Navigate to:

```
Properties → Static website hosting → Edit
```

Configure:

```text
Static website hosting      : Enable
Hosting type                : Host a static website
Index document              : index.html
Error document              : 
```

Copy the **Static Website Endpoint URL**.

---

## Section 2: Create a Route 53 Record

Go to the Route 53 console.

Create an Alias record:

```text
Subdomain   : myangular.<domainname>
Record Type : A
Alias       : Enabled
Value       : S3 bucket - myangular.<domainname>
TTL         : 60
```

---

# PART 2 - Creating a GitHub Repository and Connecting It to the Local Machine

## Section 1: Create a GitHub Repository

Sign in to GitHub.

Click **New Repository**.

Repository settings:

```text
Repository name : my-angular-project
Visibility      : Public or Private
Initialize with README : Unchecked
```

Click **Create repository**.

---

## Section 2: Clone the Repository

Copy the repository HTTPS URL.

Example:

```bash
https://github.com/<your-github-username>/my-angular-project.git
```

Clone it locally:

```bash
git clone https://github.com/<your-github-username>/my-angular-project.git
```

Move into the project directory:

```bash
cd my-angular-project
```

> **Note:** If prompted, authenticate using your GitHub credentials or a Personal Access Token (PAT).

---

## Section 3: Copy the Project Files

Open the Clarusway project folder:

```
aws/hands-on/CodePipeline_S3_AWS
```

Copy all files into the cloned **my-angular-project** folder.

Commit and push the project:

```bash
git status
git add .
git commit -m "Initial commit"
git branch -M main
git push -u origin main
```

Verify that the files appear in your GitHub repository.

---

# PART 3 - Create CodePipeline

## Step 1: Create the Pipeline

Go to **CodePipeline**.

Click **Create Pipeline**.

Pipeline settings:

```text
Pipeline name     : My First Pipeline
Service role      : Keep default
Allow CodePipeline to create a service role : Enabled
Advanced settings : Keep default
```

Click **Next**.

---

## Source Stage

```text
Source provider        : GitHub (Via GitHub App)

Connection             : Create a new connection by using "Connect to GitHub" botton
                     {P.S. we can also create the connection separately and just select it in this stage}

                     - provide a connection name
                     - install a new app (or you can select existing app if you already have)
                     - connect to your GitHub account, authenticate and select your specific repo.
                     - Click Connect. And you'll be redirected back to codepipeline page.

Repository             : my-angular-project

Branch                 : main

Output artifact format : CodePipeline default
```
Webhook events:
expand it,
Start your pipeline on push and pull request events : enabled
Event Type : Push
Filter Type : Branch
Branch : main

Hit Next.

---

## Build Stage

```text
Build provider : Other build providers 
                 AWS CodeBuild
Project Name   : Create project

    On the Create project wizard
    name                        : give name for build project
    Project type                : Default project
    Leave everything as it is,
    Operating system            : ubuntu
    Service role : create new service role
    Buildspec                   : Use a buildspec.yml file  (type : buildspec.yml)

    click continue to codepipeline , this will redirect you to the previous page.

* Make sure your prject name is selected.
leave the rest of the settings as they are.

** Click Next

```
* we can skip the test stage (Click Next)
---

## Deploy Stage

Deployment settings:

```text
Deploy provider            : Amazon S3

Region                     : N. Virginia

Bucket                     : myangular.<domainname>

S3 object key              : Leave empty

Extract file before deploy : Enabled

* expand additional configuration section :
Canned ACL                 : Public Read

Cache Control              : Leave empty
```
Click Next.
Create the pipeline.

Wait for the pipeline to complete successfully.

Browse to the S3 website endpoint to verify deployment.

---

## Step 2: Trigger the Pipeline

Open:

```
src/app/app.component.html
```

Change:

```
Version 1
```

to

```
Version 2
```

Commit and push:

```bash
git status
git add .
git commit -m "Update application version"
git push
```

Observe that CodePipeline automatically starts a new execution.

Verify that the updated website is deployed.

---

# PART 4 - Integrate Testing into CodePipeline

## Step 1: Add a Test Stage

Open:

```
src/app/calculator/calculator.component.spec.ts
```

Review the existing unit tests.

Edit your pipeline.

In the **Build** stage, click **Add Action Group** before the build action.

Configure:

```text
Action name     : Add_Test

Action provider : AWS CodeBuild

Region          : N. Virginia

Input artifact  : SourceArtifact
```

Create a second CodeBuild project.

Use the same configuration as the build project except:

```text
Buildspec : unit-test-buildspec.yml
```

Click **Continue to CodePipeline**.

Save the pipeline, click done, click save.

```

# execute the pipeline manually by using the "Release Change" button.


The pipeline should encounter some errors while executing the test stage.... why ???? ...... Permission!!

# The problem is the IAM role that we created earlier only gives permission for the first build project that we created;
# So we have to include the second project that we created for the test in the permission.


- Goto IAM roles,
- Open the IAM role that was created by the codepipeline wizard
- Under permissions section, expand codepipeline-codebuild policy
- Click Edit, json editor will open in new tab
- Under resource section, you can replace the existing project name with * wildcard, so that it will affect all projects.
- Click Next.
- Click Save Changes.

# execute the pipeline manually by using the "Release Change" button.

-* Now the pipeline should execute successfully.

---

## Step 2: Demonstrate a Failed Test

Open:

```
src/app/calculator/calculator.component.spec.ts
```

Locate:

```typescript
expect(component.result).toBe(3);
```

Change it to:

```typescript
expect(component.result).toBe(4);
```

Commit and push:

```bash
git status
git add .
git commit -m "Introduce failing unit test"
git push
```

Observe that:

- The test stage fails.
- The build stage is not executed.
- Deployment does not occur.
- The currently deployed application remains unchanged.

---

## Step 3: Fix the Test

Restore:

```typescript
expect(component.result).toBe(3);
```

Commit and push:

```bash
git status
git add .
git commit -m "Fix unit test"
git push
```

Verify that:

- The test stage passes.
- The build completes successfully.
- The deployment stage completes.
- The updated application is available.

---

# Cleanup

Delete the following resources:

- CodePipeline
- Both CodeBuild projects
- Any related EventBridge rules
- Empty and delete the artifact bucket
- Empty and delete the S3 website bucket
- (Optional) Delete the GitHub repository
