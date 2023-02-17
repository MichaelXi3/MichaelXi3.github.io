---
layout: post
title: "Firestore Many-to-many Relationship - Blog Taging"
subtitle: "Database, Firebase"
date: 2023-02-16
author: "Micheal Xi"
header-img: "img/inpost/Firestore.png"
tags: [Firestore, NoSQL, Database, Firebase]
---

# Background and Motivation

Initially, my Blog site was a simple display of all blog posts from the Firestore database collection in time order. As the number of blog posts increases, the demand for blog classification becomes more evident.

The best way of classifying blogs is by tagging them. **Each blog can have multiple tags, and each tag can correspond to multiple blogs.** Thus, this idea requires the implementation of Many-to-many relationships. So the question is: how to implement a many-to-many relationship in a NoSQL database, Firestore?

# High-level Approach

The most intuitive approach is the ****Reciprocal**** approach. The following diagram demonstrates this idea clearly.

![MTM](https://s2.loli.net/2023/02/17/ebGwrKdA6qMcvZy.png)

This diagram shows that each Tag corresponds to N blogs and each blog corresponds to N tags. Specifically, at the Firestore database level, we need two root-level collections, one is Blogs and the other is Tags. Each blog document in the Blogs collection includes a sub-collection called tags that records its multiple tags. Each tag document in the Tags collection includes a sub-collection called blogs that records the blog ids that contain this tag. Confused? Check out the structure below!

![FlowChart](https://s2.loli.net/2023/02/17/vjyLocHnBRAJNiD.png)

# Implementation

To implement this database structure, we need to make sure the standard four actions are in a good shape, including Add, Get, Delete, and Modify.

## Add Tags - Create new Blog

If we want to add a new tag to blog, we must add two documents to represent this relation. One document is the blog_id under tag collection, the other document is the tag_id under the blog document in blogs collection.

We perform this write in a `bash write`, an atomic write.

```jsx
// Add new Tags to Database - Many to many relationship
// (1) Add tags as a collection that subordinates to the blog doc
// (2) Add tags to Tags root collection as well

for await (const element of this.tagNames) {
    const blogId = this.currentBlogId; // new blog_id
    const tagId = element;             // new tag_id
    
		// The two documents that represent this relation
    const blogsRef = db.doc(`blogPosts/${blogId}/Tags/${tagId}`);
    const tagsRef = db.doc(`blogTags/${tagId}/Blogs/${blogId}`);
		
		// Atomic write
    const batch = db.batch();
    batch.set(blogsRef, {});
    batch.set(tagsRef, {});

    // Setup field value for blogTag doc to make it not empty
    const tagsDoc = db.doc(`blogTags/${tagId}`);
    batch.set(tagsDoc, { name: tagId });

		// Commit the changes
    await batch.commit();
}
```

## Get Blogs by Tag - Filter and Display

### Fetch all available Tags

To display all available tags in the Blog page, we need to fetch all tag documents in Tags collection. The implementation is:

```jsx
// Get all tags
const dataBaseTag = await db.collection("blogTags");
const dbTagResults = await dataBaseTag.get(); 
dbTagResults.forEach(doc => {
  if (this.state.allBlogTags.length === 0 || !this.state.allBlogTags.includes(doc.id)) {
    this.state.allBlogTags.push(doc.id);
  }
});
```

With that, all tags names are now stored in the allBlogTags state in Vuex.store, the state management center.

### Fetch Blogs from Database by Tags

We firstly get a list of all blog_ids that contain certain tag. Then we need to map the list of blog_ids to a list of blog documents that correspond to these ids. Finally, we push this list of blog documents (filtered by tag) to the frontend.

```jsx
// Get blogs with certain tag
async getBlogsByTag({ state }, tag ) {
  // This returns an array of blog_id that all these blogs contain a certain tag
  const blogIdsByTag = await db.collection(`blogTags/${tag}/Blogs`).get(); 

  // Get corresponding blogPost document from blogPosts collection based on blog_id
  const blogDocs = await Promise.all(
    blogIdsByTag.docs.map(doc => db.doc(`blogPosts/${doc.id}`).get())
  );
  
  // Make sure the temporary blogList by Tag is initially empty
  if (!state.blogPostByTag.length == 0) {
    state.blogPostByTag = [];
  }
	// Add Blog objects to temporary blogList by Tag
  blogDocs.forEach(doc => {
    // Object: new post
    const data = {
      blogID: doc.data().blogID,
      blogHTML: doc.data().blogHTML,
      blogCoverPhoto: doc.data().blogCoverPhoto,
      blogTitle: doc.data().blogTitle,
      blogDate: doc.data().date,
      blogCoverPhotoName: doc.data().blogCoverPhotoName,
      blogTags: doc.data().tagNames,
    };
    state.blogPostByTag.push(data);
  });
  console.log("Blogs with " + tag + " include 👇")

	// The Frontend can use this blogPostByTag list to display the blogCards filtered by tag
  console.log(state.blogPostByTag);
},
```

## Modify Tags of Blogs - Add or Delete

### Add additional Tags

The addition of new tag to existing blog is similar to the adding tags process in creating blog part.

```jsx
// This method adds tag(s) to blogPost (update)
async addTagUpdate() {
    // Error input
    if (!this.tagEnter)
        return;

    // Update database blogPost doc field tagsArray 
    var dataBase = await db.collection('blogPosts').doc(this.routeID);
    dataBase.update({
        tagNames: firebase.firestore.FieldValue.arrayUnion(this.tagEnter)
    });

    // Update database Many-to-many relationship
    const blogId = this.routeID;
    const tagId = this.tagEnter; 
    
    const blogsRef = db.doc(`blogPosts/${blogId}/Tags/${tagId}`);
    const tagsRef = db.doc(`blogTags/${tagId}/Blogs/${blogId}`);

    const batch = db.batch();
    batch.set(blogsRef, {});
    batch.set(tagsRef, {});

    // Set-up field value for blogTags if the tag is new
    const tagsDoc = db.doc(`blogTags/${tagId}`);
    batch.set(tagsDoc, { name: tagId});
    await batch.commit();

		// Empty the local input buffer
    this.tagEnter = '';
},
```

### Delete Tags from Blog

```jsx
// This method deletes tag(s) from blogPost (update)
async deleteTagUpdate() {
    // Error input
    if (!this.tagRemove)
        return;

    // Modify database blogPost doc field tagsArray 
    var dataBase = await db.collection('blogPosts').doc(this.routeID);
    dataBase.update({
        tagNames: firebase.firestore.FieldValue.arrayRemove(this.tagRemove)
    });

    // Modify database Many-to-many relationship
    const blogId = this.routeID;
    const tagId = this.tagRemove; 
    
    const blogsRef = db.doc(`blogPosts/${blogId}/Tags/${tagId}`);
    const tagsRef = db.doc(`blogTags/${tagId}/Blogs/${blogId}`);

    const batch = db.batch();
    batch.delete(blogsRef, {});
    batch.delete(tagsRef, {});
    await batch.commit();

		// Empty the local input buffer
    this.tagRemove = '';
},
```

## Delete blogPosts and Tags

```jsx
// Delete Post Action - in Vuex.store, index.js
async deletePost({ commit }, payload) {
	// Delete the blog document from Firestore database
	const blogID = payload;
	const getPost = await db.collection("blogPosts").doc(blogID);

	const batch = db.batch();
	batch.delete(getPost, {});
	await batch.commit();
		
	// Commit the Delete Changes to Frontend
	commit("filterBlogPost", payload);
},
```

Now, we get the abilities of maintaining a many-to-many relationship! We can create blogs with Tags, and filter blogs based on Tag!

# Future Work

- The ability of filtering blogs by multiple tags.
- The reciprocal approach needs 2N traversal for filtering by tag. We can improve the time complexity by the junction approach with junction table.

# References

1. Many-to-many relationship in NoSQL database: [Link](https://medium.com/firebase-tips-tricks/how-to-secure-many-to-many-relationships-in-firestore-d19f972fd4d3)
2. Firestore - Transactions and batched writes: [Link](https://firebase.google.com/docs/firestore/manage-data/transactions) 
