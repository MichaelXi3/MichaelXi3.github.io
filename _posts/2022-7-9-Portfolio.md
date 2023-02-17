---
layout: post
title: "Build Portfolio Website with Firebase and VueJS"
subtitle: "Firebase and VueJS"
date: 2022-7-9
author: "Michael Xi"
header-img: "img/inpost/VueFire.png"
tags: [VueJS, Firebase]
---
# Build Portfolio Website with Firebase and VueJS

# Overview

> Since the rise of the internet, creating a personal website has been increasingly popular. These days, the greatest method to develop a personal brand is with a website or portfolio. Many employers provide candidates the chance to submit their individual websites as a way to stand out. So, the best first project to complete if you're interested in learning about web development is to create a website for yourself.
    
> 🤖️ **Github Link**: [https://github.com/MichaelXi3/Personal_Website](https://github.com/MichaelXi3/Personal_Website)
>
> 💻 **Website Link**: [https://michaelxi.com/](https://michaelxi.com/)
> 

# Tech Stack

For my personal website, I used the combination of **VueJS**, the frontend, and the **Firebase** as my backend. 

> **Firebase** is a BaaS (Backend as a Service) company that provides the most commonly used backend services, including but not limited to user authentication, Image&Video Storage, Firestore Database, Hosting for deployment, Firebase Function, etc.
> 
> 
> ![截屏2022-07-08 下午11.22.31.png](https://s2.loli.net/2023/02/17/8ScgMLJEB3GyrjC.png)
> 

  

> **VueJS** is a very popular front-end JS framework. From my understanding, the core idea of Vue is “Build by Components”. With VueJS, you can build you feature piece by piece. For instance, the blogCard can be an individual component that used in many places. Each element is a Vue file. Inside a ‘.vue’ file, you can see HTML, JS, CSS in one page. Thus, writing vue is awesome, you can see the coordination of three basics in one page! Additionally, the core of Vue also includes “state management”, “router”, and so on.
> 
> 
> ![截屏2022-07-08 下午11.32.22.png](https://s2.loli.net/2023/02/17/8lqRtQr5kZVnfNF.png)
> 

# Project Structure

> The VueFire web project structure can be divided into Vue that responsible for the frontend and Firebase that responsible for the backend. The connection between the frontend and backend is the API provided by Firebase. For instance, the user registration can be achieved through a Firebase API with the result of storing user’s info to Firebase database.
> 

> ✍🏻 Project File Structure
>
> ![Untitled](https://s2.loli.net/2023/02/17/euyKc3GM24YipXT.png)


> **Illustration**:
> 
> - VueJS manages the structure of frontend components, each component are very likely built upon other components. For rendering components or views, the frontend needs to acquire the ‘state’ information. Thus, the ‘**State Management**’ is very crucial for a frontend framework like Vue. In Vue, it has a place called ‘store’ initiated from Vuex. The ‘store’ stores all the states used in the project, similar to the Redux of React. The only difference is that Vue’s store is much easier to implement! For the store of Vue, it connects with the Firebase database (Firestore). By functions in the store’s ‘actions’, it can interact with the data stored in Firestore. So here is a connection point between the frontend and backend. Other than that, the connections also include user authentication API, set as Admin Firebase cloud function, etc.
> - **Vue route** is another essential part of VueJS, it provides a simple way of making routes for your website. I believe VueJS has encapsulated many details behind the route section because it involves many server interactions in routes.

# Features

## Login & Register

### 🎨 Frontend

1. **Frontend Design**

> - Login, Register, Forget Password Page
> 
> ![截屏2022-07-09 上午11.38.01.png](https://s2.loli.net/2023/02/17/B4WiD3lmQFEdHMT.png)
> 
1. **VueJS Frontend HTML**

```html
<template>
  <div class="form-wrap">
      **<form** class="login">
          <p class="login-register">
              Don't have an account?
              <router-link class="router-link" :to="{ name: 'Register' }">Register</router-link>
          </p>
          <h2>Login to Explore</h2>
          <div class="inputs">
              <div class="input">
                  <input type="text" placeholder="Email" **v-model="email"** />
                  <email class="icon" />
              </div>
              <div class="input">
                  <input type="text" placeholder="Password" **v-model="password"** />
                  <password class="icon" />
              </div>
              <div v-show="error" class="error">{{ this.errorMsg }}</div>
          </div>
          <router-link class="forgot-passward" :to="{ name: 'ForgotPassword' }">Forgot your password?</router-link>
          <button @click.prevent="signIn"> Sign In </button>
          <div class="angle"></div>
      </form>
      <div class="background"></div>
  </div>
</template>
```

- Filling in information requires HTML “**form**”, we only need to decorate the default form to make it looks nicer.
- **v-model** binds the user inputs with it, so that JS script could call it by this.email and this.password.

### 🔧 Backend

<aside>
✍🏻 For Backend, we want to use user’s input email and password to create an account for the user. Then the user account information will be stored in a database with UID and user’s info. This can be achieved through calling the firebase API.

**firebase.auth()** provides the ‘user authentication’ backend service, including registration API “createUserWithEmailAndPassword”, login API “signInWithEmailAndPassword”, and forget password API “sendPasswordResetEmail”. With these encapsulated APIs, we can  build user login, register, find password feature easily.

</aside>

```jsx
methods: {
        signIn() {
            firebase
                .auth()
                .signInWithEmailAndPassword(this.email, this.password)
                .then(() => {
                    this.$router.push({ name: "Home" });
                    this.error = false;
                    this.errorMsg = "";
                    console.log("Signed in uid: " + firebase.auth().currentUser.uid);
                }).catch(err => {
                    this.error = true;
                    this.errorMsg = err.message;
                });
        }
    },
```

## Blogging

### 🎨 Frontend

> Edit Blog, View Blog
> 
> 
> ![截屏2022-07-09 下午12.12.14.png](https://s2.loli.net/2023/02/17/TSaezBIq6iALVvU.png)
> 
> ![截屏2022-07-09 下午12.22.10.png](https://s2.loli.net/2023/02/17/xvX4L5hGQMkpSfB.png)
> 

### 🔧 Backend

1. **Load all blogs to Bloggie page from Firestore**

    For each blogPost, it’s a collection of states that are stored in the Firestore database. When the web app is mounted, the data inside Firestore will be pulled out by a function called ‘getPost’ and stored in the Vuex.store (VueJS state management center). 

    Then the Blogs page will display all blogPosts based on the blogPost[] array that stores all posts states.

- **Vuex.store: Frontend State Management & Initialize States from Firebase Backend**

    ```jsx
    state: {
        // All Blogs will be populated to this array from Firestore db
        blogPost:[],
        postLoaded: null,
        
        // These are the states that a blogPost needs
        blogHTML: "Write your updates here...",
        blogTitle: "",
        blogPhotoName: "", // BlogCoverPhotoName
        blogPhotoFileURL: null,
        blogPhotoPreview: null,
        editPost: null,
    }

    **// Populate the posts from Firebase database to Firestore**
        async getPost({ state }) {
        const dataBase = await db.collection("blogPosts").orderBy('date', 'desc');
        const dbResults = await dataBase.get(); // This returns an array
        dbResults.forEach(doc => {
            // newly added doc does not already exist in blogPost array
            if (!state.blogPost.some(post => post.blogID === doc.id)) {
            // Object: new post
            const data = {
                blogID: doc.data().blogID,
                blogHTML: doc.data().blogHTML,
                blogCoverPhoto: doc.data().blogCoverPhoto,
                blogTitle: doc.data().blogTitle,
                blogDate: doc.data().date,
                blogCoverPhotoName: doc.data().blogCoverPhotoName,
            };
            state.blogPost.push(data);
            }
        });
        state.postLoaded = true;
        }
    ```

- **Firestore NoSQL Database**

    > Structure of Firestore
    > 
    > 
    > ![截屏2022-07-09 下午12.33.20.png](https://s2.loli.net/2023/02/17/TvO5JjtFzRbMqwS.png)
    > 

1. **Create a new blog post and store it in Firestore**

    Create a new Blog Post includes the process of uploading Blog cover photo, enter blog content and pictures, as well as enter blog title.

- **Upload blog cover photo**

    > Assign local data “file” with the uploaded file, the cover photo img
    > 

    ```html
    <div class="upload-file">
        <label for="blog-photo"> Upload Cover Photo </label>
        <!-- Once user uploads photo, there will be changes detected, which triggers function -->
        **<!-- 'refs' is a binding b/w user uploaded file and script method, in script portion, we can refer to this file by refs -->**
        <input @change="fileChange" type="file" ref="blogPhoto" id="blog-photo" accept=".png, .jpg, .jpeg" />
        <button @click="openPreview" class="preview" :class="{ 'button-inactive': !this.$store.state.blogPhotoFileURL }">Preview Photo</button>
        <span>File Chosen: {{ this.$store.state.blogPhotoName }}</span>
    </div>
    ```

- **Write blog content and add picture**

    > We can use vue-editor for user to input text and even images with the help of imageHandler.
    > 

    ```html
    <div class="editor">
        <vue-editor :editorOptions="editorSettings" useCustomImageHandler @image-added="imageHandler" v-model="blogHTML" />
    </div>
    ```

- **Create new post by uploading post in Firestore database & uploading images to Firebase Storage**

    > The process of create new post is simply the process of transferring user’s created states to database. As user writes a blog, the blog title is bind with v-model=”blogTitle”, and the blogHTML is bind with v-model=”blogHTML”, etc. What we need to do is to save these changes of states to our database.
    > 

    ```jsx
    uploadBlog() {
        **// Check post validation**
        if (this.blogTitle.length !== 0 && this.blogHTML.length !== 0) {
            // If the coverPhoto file is uploaded or URL is not NULL
            if (this.$store.state.blogPhotoFileURL) {
                this.loading = true;
                **// Validation Passed: Upload post cover photo to Firestore Storage
                // User Firebase Storge to store images**
                const storageRef = firebase.storage().ref(); 
                const docRef = storageRef.child(`documents/BlogCoverPhotos/${this.$store.state.blogPhotoName}`);
                docRef.put(this.file).on("state_changed", (snapshot) => {
                    console.log(snapshot);
                }, (err) => {
                    console.log(err);
                    this.loading = false;
                **// Validation Passed: Upload blog post to Firestore db**
                }, async () => {
                    const downloadURL = await docRef.getDownloadURL();
                    const timestamp = await Date.now();
                    const dateBase = await db.collection("blogPosts").doc();
                    await dateBase.set({
                        blogID: dateBase.id,
                        blogHTML: this.blogHTML,
                        blogCoverPhoto: downloadURL,
                        blogCoverPhotoName: this.blogCoverPhotoName,
                        blogTitle: this.blogTitle,
                        profileId: this.profileId,
                        date: timestamp,
                    });
                    // After press the button of publish, user will be redirected to that viewPost page to see how is't looks like
                    await this.$store.dispatch("getPost");
                    this.loading = false;
                    this.$router.push({ name: "ViewBlog", params: {blogid: dateBase.id} });
                });
                return;
            } else {
                **// Error Handling: missing Blog cover photo**
                this.error = true;
                this.errorMsg = " Please ensure you uploaded a post cover photo!";
                setTimeout(() => {
                    this.error = false;
                }, 5000);
                return;
            }
        } else {
            **// Error Handling: missing Blog content || title**
            this.error = true;
            this.errorMsg = " Please ensure Post Title & Post Content has been filled!";
            setTimeout(() => {
                this.error = false;
            }, 5000);
            return;
        }
    },
    ```

    ```jsx
    **// Image Handler of img embeded in Blog content**
    // Store the user uploaded photo to firebase storage 
    // and return back a downloadURL for the photo preview in vue-editor
    imageHandler(file, Editor, cursorLocation, resetUploader) {
        this.loading = true;
        const storageRef = firebase.storage().ref();
        const docRef = storageRef.child(`documents/blogPostPhotos/${file.name}`);
        docRef.put(file).on("state_changed", (snapshot) => {
            console.log(snapshot);
        }, (err) => {
            console.log(err);
        }, async () => {
            const downloadURL = await docRef.getDownloadURL();
            **// Insert img preview in vue-editor**
            Editor.insertEmbed(cursorLocation, "image", downloadURL);
            this.loading = false;
            resetUploader();
        })
    }
    ```

## Set Admin

### 🔧 Backend

- The special part of set user as Admin is that it involves the Firebase Cloud Function! Check out its definition below

- Cloud Functions for Firebase is **a serverless framework that lets you automatically run backend code in response to events triggered by Firebase features and HTTPS requests.


1. **Call addAdmin Firebase cloud function at Frontend**

    ```jsx
    async addAdmin() {
        // Connect to firebase cloud functions
        const addAdmin = await **firebase.functions().httpsCallable('addAdminRole')**;
        // **Pass parameters** to firebase cloud functions
        const result = await addAdmin({ email: this.adminEmail });
        // Set functionMsg to display at span, HTML
        this.functionMsg = result.data.message;
    }
    ```

2. **Firebase cloud function implementation**

    > The cloud function of Firebase is a **`Node.JS function`**
    > 

    ```jsx
    const functions = require("firebase-functions");
    const admin = require("firebase-admin")
    admin.initializeApp();

    // Cloud Function: addAdminRole
    // Implementations: multiple callbacks
    // After finished function, you need to deploy it to the firebase function by 'firebase  deploy --only functions' at terminal
    exports.addAdminRole = **functions.https.onCall((data, context)** => {
    return admin
        .auth()
        .getUserByEmail(data.email) // Identify user by user's email
        .then((user) => {
            **// This step adds a new admin setting to user's claim**
            return admin.auth().**setCustomUserClaims**(user.uid, {
                admin: true,
            });
        // Success Notification
        }).then(() => {
            console.log("Successfully Added!");
            return {
                message: `Success! ${data.email} has been made an admin!`,
            };
        })
        // Error Notification
        .catch((err) => {
            console.log("Error");
            console.log(err);
        });
    });
    ```

- **Check if a user is an Admin**

    ```jsx
    **// Get user's token or user's claims info**
    const token = await user.getIdTokenResult();
    const admin = await token.claims.admin; // Boolean
    commit('setProfileAdmin', admin); // Set curUser as admin if admin=true
    ```

# Deployment

> Firebase made deployment easy than ever. We only need to take the advantage of “**Hosting**” service of firebase.
> 
> 
> ![截屏2022-07-09 下午1.59.14.png](https://s2.loli.net/2023/02/17/Fs3v6RU8rbAuEca.png)
> 

After you finished all implementations of your website, use the following command to get ready for deployment: `npm run build`. This command with generate a file called “dist” that will be deployed to Firebase server. It’s a compressed file that can be recognized and rendered. Then, with the firebase CLI (command line interface) installed, enter `firebase deploy`. Congratulations! You website is now launched to the internet!

```bash
npm run build
firebase deploy
```

## Customize Domain Name

> Firebase will give you a domain name like “**[personalweb-55e41.web.app](https://personalweb-55e41.web.app/)**” which is hard to memorize. Thus, it’s better to cutomize our domain name. To do this, we firstly need to select a Domain name provider. I chosed Google Domain but you also can choose NameCheap, etc.Then you can buy and customize your domain name!

After you choose a Domain name address and connect it to Firebase hosting. Firebase hosting will redirect the original domain to your new domain. Finally, everything works! In the world of internet, the domain is managed in DNS (Domain Name System), where there’re many DNS servers that constantly matching website domain names to corresponding IP adress so that your website can be found and display. So thank you DNS server!

❤️ Final word: welcome everyone to visit my site: [michaelxi.com](http://michaelxi.com)
>