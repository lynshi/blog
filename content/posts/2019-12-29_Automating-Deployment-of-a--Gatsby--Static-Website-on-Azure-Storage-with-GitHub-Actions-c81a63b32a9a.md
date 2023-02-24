---
title: >-
  Automating Deployment of a (Gatsby) Static Website on Azure Storage with
  GitHub Actions
description: >-
  Read on to learn how to deploy your personal website to Azure Storage on every
  push to GitHub.
date: '2019-12-29T19:46:39.239Z'
slug: >-
  /@shilyndon/automating-deployment-of-a-gatsby-static-website-on-azure-storage-with-github-actions-c81a63b32a9a
categories:
  - Deployment
tags:
  - Gatsby
  - Microsoft Azure
  - Automation
  - Website
---

![](/img/medium/1__HbQtNXVua60jpqXmAgM4Xg.png)

I host my own static website on Azure Storage, and I got tired of running a script to upload my content every time I made a change to my website. After some Googling, I discovered [GitHub Actions](https://github.blog/2019-08-08-github-actions-now-supports-ci-cd/), made public about a month ago, which lets users automate how they build, test, and deploy their projects. Interestingly, though I found great guides for [automating static website deployment to Azure using GitLab](https://medium.com/faun/automating-your-deployment-using-gitlab-azure-storage-static-website-hosting-75c767b2569f) and [building and deploying Gatsby sites with GitHub Actions](https://nehalist.io/building-and-deploying-gatsby-sites-with-github-actions/), there wasn’t one guide with everything I needed, and I had to cobble pieces together from different sources. This is an attempt to rectify that.

This post assumes your static website is already up and running on Azure Storage. If not, the [Azure documentation on static website hosting](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-static-website) was enough for me to get started.

#### Setup

To begin, go to the Azure Storage account with your static website content, and find the setting “Access keys”. Copy either one of the connection strings; this is needed to upload content to Azure via GitHub actions.

![](/img/medium/1__nvKSMIPWQYfbcgUenfgJpg.png)

Now, open your personal website repository on GitHub. Go to the settings tab and select “Secrets” in the left panel. Click “Add a new secret”, name it “AZURE\_STORAGE\_CONNECTION\_STRING” (you can use a different name as long as you update the GitHub Actions `.yml` accordingly), and paste the connection string value obtained from Azure into the “Value” area.

![](/img/medium/1__MlelyBz__fvwnVyp99E6dpQ.png)

#### Workflow Script

To build your Action, go to the “Actions” tab in your repository. Select “New workflow” and use the `Node.js` starter.

![](/img/medium/1__dgJ8CQoCfQNh__q7VNRgPeQ.png)

In the page that loads, paste the following code into the editor.

The above `.yml` is tailored to my personal website, which was built with [React](https://reactjs.org/) and [Gatsby](https://www.gatsbyjs.org/). Hence, you may not need all the steps listed.

1.  The `Install dependencies` step is only necessary if you used [Node](https://nodejs.org/en/) packages.
2.  The `Build with Gatsby` step can be replaced with your build command, if you have one.
3.  The `Azure upload` step is the only required step. The first line in the script deletes all your old files in Azure, while the second uploads your local files. Since Gatsby generates my website files in the `public` folder, I upload that (`-s` parameter); you may need a different value there.

After uploading your files, you may need to [purge your CDN endpoint](https://docs.microsoft.com/en-us/azure/cdn/cdn-purge-endpoint) to see the updates. (Update: I now [use GitHub actions to automate purging](https://medium.com/@shilyndon/purging-azure-cdn-with-github-actions-1c18e2adaf18?source=friends_link&sk=0e0936d055d9d5cfcb5f4847af0445c0).)

That’s it! Now, your content will automatically be uploaded to Azure Storage on every push.