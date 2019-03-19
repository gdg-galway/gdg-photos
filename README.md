# Your own "Photos" App #

The objective of this tutorial is to build a simple Photos application based on Google Photos or Apple Photos. This app will allow the user to login, upload photos, and will automatically sort pictures based on their content, date of creation or GPS coodinates where the picture was taken.

To do so we will use multiple Firebase services such as Hosting, Storage, Authentication, Cloud Functions and Firestore. We will also use the Google Cloud Vision API to analyze the user's photos and determine their content.

## Required Dependencies ##

- NodeJS 8+ (https://nodejs.org/en/)
- Firebase CLI (https://github.com/firebase/firebase-tools)

## Languages/Syntax/Preferences ##

We will use HTML5, CSS3 and JavaScript (ESNext) in this tutorial. The code is not production-ready but can be passed through Webpack/Parcel/Babel for max compatibility.

Note that Google Cloud Functions can be written is JavaScript, TypeScript, Golang or Python.

## Setup the Project ##

You can start by cloning this repository.

You will need to create a Google account if you don't have one, and register on Google Cloud Platform (https://console.cloud.google.com). You can get $300 credits for free to test the platform.

Now, head over to the Firebase console to create a new Firebase project: https://console.firebase.google.com.

Once you have done that, open your terminal and install the Firebase CLI module if it is not already done by using the following command `npm install -g firebase-tools`.

Browse to the repository folder before typing the following command `firebase init`. If you're using Firebase for the first time you will have to login using `firebase login`. You will have to select the project you have just created from your list of Firebase projects. You will have to select the Firebase services that you will be using in your project: Firestore, Storage, Functions and Hosting. Pick the default options for the rest. A prompt will ask you if you want to replace the `index.html` file already in your `public` folder, say `NO (n)`.

Because we will be using Google Cloud Vision API, you will have to enable the API in the Google Cloud Project created by Firebase on https://console.cloud.google.com.

## The App ##

For this tutorial, I have included the HTML/CSS/JS file `public/index.html` that creates the basic UI of this application so you don't have to bother with that and you can focus on the features that matter.

### Authentication ###

We will be using two ways to authenticate our users: Email/Password and via their Google account. To be able to do so, you will have to enable these methods in your Firebase project dashboard in the Authentication section under the Sign-in methods tab.

The code necessary to sign-up/login/logout is minimal. We create the 4 functions following:

```js
async function createAccount(e) {
    e.preventDefault()
    setLoading(true)
    try {
        // Pass the email address + password to create an account
        await auth.createUserWithEmailAndPassword(signUpEmail.value, signUpPassword.value)
        signUpEmail.value = signUpPassword.value = ''
    }
    catch(e) {
        alert(e.message)
        setLoading(false)
    }
}

async function login(e) {
    e.preventDefault()
    setLoading(true)
    try {
        // Pass the email address + password to authenticate
        await auth.signInWithEmailAndPassword(loginEmail.value, loginPassword.value)
        loginEmail.value = loginPassword.value = ''
    }
    catch(e) {
        alert(e.message)
        setLoading(false)
    }
}

async function loginWithGoogle() {
    setLoading(true)
    // Instantiate Google Authentication
    const provider = new firebase.auth.GoogleAuthProvider()
    try {
        // Start the Google user authentication flow
        await auth.signInWithPopup(provider)
    }
    catch(e) {
        console.log('ERROR_LOGIN_WITH_GOOGLE', e.message)
        setLoading(false)
    }
}

function logout() {
    auth.signOut()
    user = null
    document.body.classList.remove('connected', 'init')
    setLoading(false)
}
```

It will be useful to not only have our user data available on Firebase Storage but also to store their data in our Firestore database so we can add more data to their profile if we need to. To automatically add their account to the database, we will write our first Cloud Function using the `onCreate` trigger available in Firebase Authentication.

Let's open our `index.js` file available in your `functions` directory.

First, we will have to include Firebase Admin which is the node module that will allow you to run Firebase from your backend. Firebase Admin is already pre-installed, so you just have to include it in your `index.js` file and initialize the library. Note that you don't have to pass any parameters when initializing Firebase Admin since you will be running it within the Cloud Functions environment.

```js
const firebase = require('firebase-admin')

firebase.initializeApp()

// Shorthand for our Firestore database
const db = firebase.firestore
// Shorthand for our Cloud Storage bucket
const bucket = firebase.storage().bucket()
```

We can now create our function. Note that Firestore is a No-SQL document-based database. Documents are stored in Collections the same way Rows are stored in Tables. A document is a simple JSON object and can be nested if you want it to. A document can have Subcollections, for example the document related to a User could have a subcollection called Friends.

Because we will be using Firestore, you will have to enable and create your database in your Firebase console in the Database section.

```js
// To add a function, extend the "exports" object
exports.userCreated = functions.auth.user().onCreate(user => {
    return db().collection('users').doc(user.uid).set({
        id: user.uid,
        name: user.displayName || '',
        email: user.email || '',
        created_at: db.FieldValue.serverTimestamp(),
        updated_at: db.FieldValue.serverTimestamp(),
    })
})
```

Note that our database is locked at the moment. This means that only admin accounts can read and write in the database. In our case, we want authenticated users to be able to read their data and access their photos. To do so, we will have to edit our `firestore.rules` file.

```
service cloud.firestore {
    match /databases/{database}/documents {
        match /users/{userId} {
            allow read: if request.auth.uid == userId;
        }
        match /photos/{photoId} {
            allow read: if resource.data.user_id == request.auth.uid;
        }
        match /{document=**} {
            allow read, write: if false;
        }
    }
}
```

Now let's deploy our changes using the `firebase deploy` command.

We'll go back to our `index.html` file, and get our user data from Firestore on the front-end.

We will use the `onAuthStateChanged` function to detect when a user connects or disconnects, and get their data along with their photos from Firestore by using the `onSnapshot` method. This method is triggered every time the user document is updated in the database.

```js
auth.onAuthStateChanged(userData => {
    console.log('USER_DATA', userData)
    // Empty photos container
    photosContainer.innerHTML = ''
    // If a Snapshot instance is already running, we stop it
    if(userRef) userRef()
    if(photosRef) photosRef()
    // userData is null if the user is not connected
    if(userData) {
        // If connected, we get the user document
        userRef = db().collection('users').doc(userData.uid).onSnapshot(doc => {
            if(doc.exists) {
                user = doc.data()
                console.log('USER', user)
                // If the user exists in our database, we get the photos uploaded by this user
                photosRef = db().collection('photos').where('user_id', '==', doc.id).orderBy('created_at', 'desc').onSnapshot(results => getPhotos(results))
                document.body.classList.add('connected')
                document.body.classList.remove('init')
                setLoading(false)
            }
        })
    }
    else document.body.classList.remove('init')
})
```

### Upload photos with Firebase Storage ###

The same way we have setup Firestore, we'll have to setup and secure Fibase Storage. Let's open the Firebase console again and head over to the Storage section where you will have to enable Storage.

Now, we'll open our `storage.rules` file and update our rules so that authenticated users can access their pictures and upload images.

```
service firebase.storage {
    match /b/{bucket}/o {
        match /users/{userId}/photos/{photo} {
            allow read: if userId == request.auth.uid;
            // write if folder belongs to authenticated user and if file is an image under 10MB
            allow write: if request.resource.size < 10 * 1024 * 1024
                        && request.resource.contentType.matches('image/.*')
                        && userId == request.auth.uid;
        }
    }
}
```

And we'll deploy our rules using `firebase deploy`.

Now, we can head back to our `index.html` file to implement a method that will allow our users to upload images. We will pass a couple of custom metadata with our file like ID and filename so that we can use them later.

```js
async function uploadFiles() {
    const files = Array.from(fileInput.files)
    if(files.length) {
        setLoading(true)
        try {
            // Upload multiple files at once
            await Promise.all(files.map(file => {
                // Generate a random ID
                const id = db().collection('photos').doc().id
                const ext = file.name.split('.').pop()
                return storage.ref(`users/${user.id}/photos/${id}.${ext}`).put(file, {
                    customMetadata: {
                        id,
                        name: file.name
                    }
                })
            }))
        }
        catch(e) {
            console.log(e)
        }
        setLoading(false)
    }
    fileInput.value = ''
}
```

## Process and analyze images ##

In order to sort/categorize pictures like other Photos apps do, we'll need to know the date of creation of the photo and where it was taken. Luckily, today most phones and cameras encode this information in the EXIF metadata of each photo.

We will need a few Node modules first, use the following command in your `functions` folder to install them `npm install -S fs-extra child-process-promise exif @google/maps @google-cloud/vision`.

### Extract the EXIF metadata ###

There's a nice NPM module called `exif` that parse and return the EXIF data of a given picture. Let's add that to our `index.js` file.

```js
const {ExifImage} = require('exif')

const getImageExif = (image) => {
    return new Promise((resolve, reject) => {
        new ExifImage({ image }, (error, data) => {
            if(error) reject(error)
            else resolve(data)
        })
    })
}
```

This should give us access to the GPS coordinates where the picture was taken. We just need to convert them into latitude and longitude coordinates.

```js
const getCoodinates = gps => {
    const latitude = (gps.GPSLatitude[0] + (gps.GPSLatitude[1] / 60) + (gps.GPSLatitude[2] / 3600)) * (gps.GPSLatitudeRef === 'N' ? 1 : -1)
    const longitude = (gps.GPSLongitude[0] + (gps.GPSLongitude[1] / 60) + (gps.GPSLongitude[2] / 3600)) * (gps.GPSLongitudeRef === 'E' ? 1 : -1)
    return {latitude, longitude}
}
```

We can pass the coordinates we now have to the Google Maps API to get the information we need like city and country. You will need to enable this API in your Google Cloud Project and get an API Server Key in order to use it.

```js
const mapsClient = require('@google/maps').createClient({ Promise: Promise, key: 'YOUR_SERVER_KEY' })

const getLocation = async gps => {
    const res = await mapsClient.reverseGeocode({ latlng: `${gps.latitude},${gps.longitude}` }).asPromise()
    const address = res.json.results[0].address_components
    const city = address.find(component => component.types.includes('locality') || component.types.includes('administrative_area_level_1')) || null
    const country = address.find(component => component.types.includes('country')) || null
    return { city, country }
}
```

We also need to extract the date of creation of the picture.

```js
const getCreateDate = date => {
    if(!date) return new Date()
    const split = date.split(' ')
    return new Date(`${split[0].replace(/:/g, '/')} ${split[1]}`)
}
```

Our users will be uploading large photos, 4k probably, we shouldn't load the full picture as thumbnail every time they want to have a look at their albums. We'll have to generate our thumbnails.

The instances running Cloud Functions have ImageMagick preinstalled. This library allows you to manipulate images to create thumbnails and optimize images in general.

Here are a few functions that we will use to create a 300x300 thumbnail and to generate a GIF placeholder 3x3 to show while the thumbnail is loading.

```js
const fs = require('fs-extra')
const {spawn, exec} = require('child-process-promise')

const getImagePlaceholder = async (tmp) => {
    try {
        // Here is how we generate our 3x3 GIF that we'll use as placeholder while our thumbnails load
        const base64 = await exec(`convert ${tmp} -resize 3x3 INLINE:GIF:-`)
        return base64.stdout
    }
    catch(err) {
        return ''
    }
}

const convertImage = async (path, destination, width, height) => {
    // This code generates our thumbnail
    return spawn('convert', [
        path,
        '-filter', 'Triangle',
        '-define', 'filter:support=2',
        '-thumbnail', `${width}x${height}>`,
        '-unsharp', '0.25x0.08+8.3+0.045',
        '-dither', 'None',
        '-posterize', '136',
        '-quality', '82',
        '-define', 'png:compression-filter=5',
        '-define', 'png:compression-level=9',
        '-define', 'png:compression-strategy=1',
        '-define', 'png:exclude-chunk=all',
        '-interlace', 'none',
        '-colorspace', 'sRGB',
        '-strip',
        destination
    ])
}

const getPublicUrl = (path) => {
    // We generate our Google Storage link based on our bucket name and path to our file on GCS
    return `https://storage.googleapis.com/${bucket.name}/${encodeURIComponent(path)}?v=${Date.now()}`
}

const optimizeImage = async (path, destination, metadata = {}, width = 1280, height = 720) => {
    const thumbnail = path + '_o'
    try {
        await convertImage(path, thumbnail, width, height)
    }
    catch(err) {
        console.log('OPTIMIZATION_ERROR', err)
        throw err
    }
    try {
        // This function uploads our thumbnail to GCS and makes it public
        await bucket.upload(thumbnail, {
            destination,
            public: true,
            resumable: false,
            metadata
        })
    }
    catch(err) {
        console.log('UPLOAD_ERROR', err)
        await fs.unlink(thumbnail)
        throw err
    }
    const base64 = await getImagePlaceholder(path)
    await fs.unlink(thumbnail)
    return {
        url: getPublicUrl(destination),
        placeholder: base64,
    }
}
```

Now let's put all our pieces together to create a new Cloud Function that will be triggered every time a new photo is uploaded. We will also include the code that will get information on the content of the image from Google Cloud Vision API.

```js
const vision = require('@google-cloud/vision')

const imageAnnotator = new vision.ImageAnnotatorClient()

// We boost the max memory that our function can use to 1GB (128MB by default)
exports.fileUploaded = functions.runWith({ memory: '1GB' }).storage.object().onFinalize(async (data) => {
    const metadata = data.metadata || {}
    // We don't run the rest of the code if the image was updated or if the image was already processed
    if(data.metageneration !== '1' || metadata.optimized) return false
    console.log('FILE_DATA', data)
    // We make sure that the image is in a photo folder and associated with one of our users
    const photoData = data.name.match(/^users\/([a-zA-Z0-9]+)\/photos\/([a-zA-Z0-9]+).([a-zA-Z0-9]+)$/)
    if(photoData) {
        const id = photoData[2]
        const destination = `/tmp/${id}`
        // Download the picture from GCS
        await bucket.file(data.name).download({ destination })
        let exif
        try {
            // Extract EXIF data
            exif = await getImageExif(destination)
        }
        catch(e) {
            console.log('EXIF_ERROR', e)
        }
        // Generate the 300x300 thumbnail along with the 3x3 GIF placeholder
        const thumbnail = await optimizeImage(destination, `users/${photoData[1]}/photos/${id}_300x300.${photoData[2]}`, {
            contentType: data.contentType,
            metadata: Object.assign(metadata, { optimized: true })
        }, 300, 300)
        // Cleanup
        await fs.unlink(destination)
        // Get coordinates
        const gps = exif.gps && exif.gps.GPSLatitude && exif.gps.GPSLongitude ? getCoodinates(exif.gps) : null
        let location = null
        if(gps) {
            try {
                // Get location from Google Maps API
                location = await getLocation(gps)
            }
            catch(e) {
                console.log('GEOCODE_ERROR', e)
            }
        }
        let annotations = null
        try {
            // Get annotations like Faces, Objects, Places, Landmarks from the photo
            [annotations] = await imageAnnotator.annotateImage({
                image: {source: {imageUri: `gs://${bucket.name}/${data.name}`}},
                features: [
                    {type: 'FACE_DETECTION', maxResults: 5},
                    {type: 'LABEL_DETECTION', maxResults: 5},
                    {type: 'LANDMARK_DETECTION', maxResults: 5},
                ],
            })
        }
        catch(e) {
            console.log('ANNOTATOR_ERROR', e)
        }
        console.log('ANNOTATIONS_DATA', annotations)
        const photo = {
            id,
            name: metadata.name || '',
            user_id: photoData[1],
            type: data.contentType || `image/${photoData[3]}`,
            gps,
            location,
            thumbnail: thumbnail.url,
            placeholder: thumbnail.placeholder,
            url: `https://firebasestorage.googleapis.com/v0/b/${bucket.name}/o/${encodeURIComponent(data.name)}?alt=media&token=${metadata.firebaseStorageDownloadTokens}`,
            width: exif.exif.ExifImageWidth,
            height: exif.exif.ExifImageHeight,
            annotations,
            labels: annotations ? annotations.labelAnnotations.map(label => label.description) : [],
            created_at: getCreateDate(exif.exif.CreateDate),
            uploaded_at: db.FieldValue.serverTimestamp(),
        }
        console.log('PHOTO', photo)
        // Add our picture to Firestore in the 'photos' collection
        return db().collection('photos').doc(id).set(photo)
    }
    return true
})
```

Now let's deploy our code one last time using `firebase deploy`. You should also be able to access, test and share your app securely (https) on the web via Firebase Hosting.

## Conclusion ##

And here we are! We have all we need to categorize and sort our photos. Location, content, date, people, everything we need to sort pictures depending on their context and create albums based on that exactly like other Photos apps out there.

I will leave the rest of it to you. You can add a lot more features to this tutorial. An example would be to use the facial landmarks returned by the Vision API to recognize people from photos and create albums based on who's in the picture! Here's an article to learn more on face recognition: https://medium.com/@ageitgey/machine-learning-is-fun-part-4-modern-face-recognition-with-deep-learning-c3cffc121d78.

I hope you had a good time going through this tutorial.
If you have any questions, feel free to contact us on Twitter or Discord:

- Twitter: https://twitter.com/GDGgalway
- Discord: https://discord.gg/JWNVT4W