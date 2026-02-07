---
title: "How to Create a Chrome Extension to Retrieve URLs of All Open Tabs"
description: Build a simple Chrome extension to get and display URLs of all your open browser tabs
author: Vaibhav Gagneja
date: 2025-03-30 15:27:15 +0000
categories: [Development, Chrome Extensions]
tags: [chrome-extension, javascript, tutorial, web-development]
---

## Introduction

Have you ever needed to **get the URLs of all your open tabs in Chrome**? Whether you're a developer, researcher, or just a heavy browser user, having a simple way to retrieve and manage your open tab URLs can be incredibly useful. In this tutorial, I'll walk you through creating a **Chrome extension** that fetches and displays the URLs of all open tabs.

## Requirements

Before we start, ensure you have the following:

* A basic understanding of HTML, JavaScript, and JSON.
* Google Chrome installed on your computer.

## Step-by-Step Guide

### Step 1: Set Up Your Project

* **Create the Project Folder:** Create a new folder on your computer. Name it something like `GetTabsURLs`.

* **Create `manifest.json`**: Inside the `GetTabsURLs` folder, create a file named `manifest.json` and add the following content:

```json
{
  "manifest_version": 3,
  "name": "Get Open Tabs URLs",
  "version": "1.0",
  "permissions": ["tabs"],
  "background": {
    "service_worker": "background.js"
  },
  "action": {
    "default_popup": "popup.html"
  }
}
```

**What each field means:**

* `manifest_version`: Version 3 is the latest and most current.
* `name`: The name of your extension that users will see.
* `version`: Helps in managing updates.
* `permissions`: Asks for permission to access browser tabs.
* `background`: Specifies the background script that runs behind the scenes.
* `action`: Defines the UI of your extension's icon.

### Step 2: Create the Background Script

**Create `background.js`**: In the same folder, create a file named `background.js` and add the following content:

```javascript
chrome.action.onClicked.addListener((tab) => {
  chrome.tabs.query({}, (tabs) => {
    let urls = tabs.map(tab => tab.url);
    console.log(urls);
    alert(urls.join('\n'));
  });
});
```

**How it works:**

* `chrome.action.onClicked.addListener`: Listens for clicks on the extension icon.
* `chrome.tabs.query({}, (tabs) => {...})`: Queries all open tabs in the browser.
* `tabs.map(tab => tab.url)`: Extracts the URL from each tab object.
* `console.log(urls)`: Logs the URLs to the console for debugging.
* `alert(urls.join('\n'))`: Displays the URLs in a pop-up alert.

### Step 3: Create the Popup Interface

**Create `popup.html`**: In the `GetTabsURLs` folder, create a file named `popup.html` with the following content:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Get Open Tabs URLs</title>
  </head>
  <body>
    <button id="getTabs">Get Tabs URLs</button>
    <pre id="output"></pre>
    <script src="popup.js"></script>
  </body>
</html>
```

**Elements explained:**

* `<button id="getTabs">`: A button users will click to fetch the URLs.
* `<pre id="output">`: Where the URLs will be displayed in preformatted text style.
* `<script src="popup.js">`: Links to the JavaScript file that handles button clicks.

### Step 4: Create the Popup Script

**Create `popup.js`**: Finally, create a file named `popup.js` with the following content:

```javascript
document.getElementById('getTabs').addEventListener('click', () => {
  chrome.tabs.query({}, (tabs) => {
    let urls = tabs.map(tab => tab.url);
    document.getElementById('output').textContent = urls.join('\n');
  });
});
```

**How it works:**

* `document.getElementById('getTabs')`: Selects the button element by its ID.
* `.addEventListener('click', ...)`: Adds a click event listener to the button.
* `chrome.tabs.query({}, (tabs) => {...})`: Queries all open tabs.
* `tabs.map(tab => tab.url)`: Maps the tabs to their URLs.
* Updates the `pre` element with the list of URLs, separated by new lines.

### Step 5: Load and Test Your Extension

**Load the Extension in Chrome:**

1. Open Chrome and navigate to `chrome://extensions/`
2. Enable `Developer mode` by toggling the switch in the top right corner
3. Click the `Load unpacked` button and select the `GetTabsURLs` folder

**Use the Extension:**

1. After loading the extension, you should see its icon appear in the toolbar
2. Click on the extension icon to open the popup
3. In the popup, click the `Get Tabs URLs` button
4. The URLs of all open tabs will be displayed in the `pre` element

---

## Optional: Adding an Icon

If you want to add an icon to your extension:

1. **Create an `icons` folder**: Inside your `GetTabsURLs` folder, create a new folder named `icons`.

2. **Add Icon Images**: Add your icon images (e.g., `icon.png`) in different sizes (16x16, 48x48, 128x128).

3. **Update `manifest.json`**: Update your `manifest.json` to include the icons:

```json
{
  "manifest_version": 3,
  "name": "Get Open Tabs URLs",
  "version": "1.0",
  "permissions": ["tabs"],
  "background": {
    "service_worker": "background.js"
  },
  "action": {
    "default_popup": "popup.html",
    "default_icon": {
      "16": "icons/icon16.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    }
  },
  "icons": {
    "16": "icons/icon16.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  }
}
```

---

## Summary

| File | Purpose |
|------|---------|
| `manifest.json` | Sets up the extension's basic information and permissions |
| `background.js` | Handles clicks on the extension icon and retrieves tab URLs |
| `popup.html` | Creates the visual layout of the popup |
| `popup.js` | Manages interactions within the popup |

By understanding each part of the code, you'll be able to modify and enhance the extension to fit your needs. Happy coding!

You can find the full code on [GitHub](https://github.com/VaibhavGagneja/GetChromeUrlsExtention).
