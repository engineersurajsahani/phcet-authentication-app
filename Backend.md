# 🔐 Auth App API (Backend)

Node.js + Express se bana ek simple **Authentication API** jisme Firebase Firestore database use hua hai. Isme **Register** aur **Login** ke endpoints hain, password **bcryptjs** se hash hota hai, aur login par **JWT token** generate hota hai.

---

## 📁 Project Structure

```
auth-app-api/
├── server.js              # Main server file (entry point)
├── package.json           # Project dependencies
├── serviceAccountKey.json # Firebase credentials (⚠️ isko kabhi share/commit mat karo)
├── config/
│   └── db.js              # Firebase Firestore connection
├── models/
│   └── User.js            # User model (DB queries)
└── router/
    └── authRouter.js      # Register & Login API endpoints
```

---

## ⚙️ Dependencies (package.json)

```json
{
  "name": "auth-app-api",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "bcryptjs": "^3.0.3",
    "cors": "^2.8.6",
    "express": "^5.2.1",
    "firebase-admin": "^14.4.0",
    "jsonwebtoken": "^9.0.3"
  }
}
```

**Explanation:**
- **express** → Web framework jo API endpoints banane ke liye use hota hai
- **cors** → Cross-Origin requests allow karta hai (Flutter app alag port par chalti hai, isliye zaroori hai)
- **bcryptjs** → Password ko securely hash karne ke liye
- **jsonwebtoken (jwt)** → Login ke baad token generate karne ke liye
- **firebase-admin** → Firebase Firestore database se connect karne ke liye

**Install karne ke liye:**
```bash
npm install
```

---

## 🚀 server.js (Entry Point)

```js
const express=require('express');
const db=require('./config/db');
const authRouter=require('./router/authRouter');
const cors=require('cors');

const app=express();

app.use(cors('*'));
app.use(express.json());
app.use('/auth',authRouter);

const requestLogger=(reuqest,response,next)=>{
    console.log(`Request Method :- ${request.method} , Request URL :- ${request.url} , Date & Time :- ${new Date().toLocaleString()}`);
    next();
}
app.use(requestLogger);

app.listen(4000,()=>{
    console.log("Server is running on port 4000");
});
```

**Explanation:**
1. `express`, `db` (Firebase config), `authRouter` aur `cors` ko import kiya gaya hai
2. `app.use(cors('*'))` → Sabhi origins se requests allow karta hai (development ke liye theek hai)
3. `app.use(express.json())` → JSON request body ko parse karta hai, isse `request.body` me data milta hai
4. `app.use('/auth', authRouter)` → Saare auth routes `/auth` prefix ke saath mount hote hain (jaise `/auth/register`, `/auth/login`)
5. `requestLogger` → Ek simple middleware jo har request ka method, URL aur time console par print karta hai, phir `next()` se request aage badhti hai
6. `app.listen(4000)` → Server port **4000** par start hota hai

**Run karne ke liye:**
```bash
node server.js
```

---

## 🔥 config/db.js (Firebase Firestore Connection)

```js
const {initializeApp,cert}=require('firebase-admin/app');
const {getFirestore}=require('firebase-admin/firestore');
const serviceAccountKey=require('../serviceAccountKey.json');

initializeApp({
    credential:cert(serviceAccountKey)
});

const db=getFirestore();

module.exports=db;
```

**Explanation:**
1. `initializeApp` aur `cert` firebase-admin se import kiye — Firebase app initialize karne ke liye
2. `getFirestore` se Firestore database ka instance milta hai
3. `serviceAccountKey.json` → Firebase Console se download hota hai (Project Settings → Service Accounts → Generate New Private Key). Isme database access ki secret credentials hoti hain
4. `initializeApp({credential: cert(serviceAccountKey)})` → Firebase app ko service account credentials ke saath initialize karta hai
5. `db = getFirestore()` → Firestore ka instance banaya aur export kar diya taaki baaki files use kar sakein

⚠️ **Important:** `serviceAccountKey.json` ko `.gitignore` me daalna chahiye — ye highly sensitive file hai.

---

## 👤 models/User.js (User Model — DB Queries)

```js
const db=require('../config/db');

class User{

    static async findUserByUsername(username){
        const snapshot=await db.collection('users').where('username','==',username).limit(1).get();
        if(snapshot.empty){
            return null;
        }
        const doc=snapshot.docs[0];
        return {id:doc.id,...doc.data()};
    }

    static async findUserByEmail(email){
        const snapshot=await db.collection('users').where('email','==',email).limit(1).get();
        if(snapshot.empty){
            return null;
        }
        const doc=snapshot.docs[0];
        return {id:doc.id,...doc.data()};
    }

    static async register(user){
        const docRef=await db.collection('users').add(user);
        const doc=await docRef.get();
        if(!doc.exists){
            return null;
        }
        return {id:doc.id,...doc.data()};
    }
}

module.exports=User;
```

**Explanation:**
- Ye ek static class hai jisme users collection par queries likhi gayi hain
- **`findUserByUsername(username)`** → `users` collection me wo document dhundta hai jiska `username` match karta hai. `limit(1)` se sirf 1 result aata hai. Agar nahi mila to `null` return hota hai, warna document ka `id` + data return hota hai
- **`findUserByEmail(email)`** → Same logic, bas `email` field par query karta hai (register ke time duplicate email check karne ke liye)
- **`register(user)`** → Naya user document `users` collection me `add()` se insert karta hai. Firestore khud ek unique `id` generate karta hai. Insert hone ke baad document wapas fetch karke `{id, ...data}` return karta hai

---

## 🛣️ router/authRouter.js (API Endpoints)

```js
const express=require('express');
const User=require('../models/User');
const bcrypt=require('bcryptjs');
const jwt=require('jsonwebtoken');

const router=express.Router();

router.post('/register',async (request,response)=>{
    try {
        const {name,username,email,password}=request.body;

        if(!name){
            return response.status(400).json({message:"Name Field Is Required!!!"});
        }
        if(!username){
            return response.status(400).json({message:"Username Field Is Required!!!"});
        }
        if(!email){
            return response.status(400).json({message:"Email Field Is Required!!!"});
        }
        if(!password){
            return response.status(400).json({message:"Password Field Is Required!!!"});
        }

        const existingUsername=await User.findUserByUsername(username);
        if(existingUsername){
            return response.status(400).json({message:"Username Already Exists!!!"});
        }

        const existingEmail=await User.findUserByEmail(email);
        if(existingEmail){
            return response.status(400).json({message:"Email Already Exists!!!"});
        }

        const hashPassword=await bcrypt.hash(password,10);

        const newUser={
            name,
            username,
            email,
            password:hashPassword
        };

        const user=await User.register(newUser);

        if(!user){
            return response.status(500).json({message:"Failed To Register New User"});
        }

        response.status(201).json({message:"User Created Successfully!!!"});
    } catch (error) {
        response.status(500).json({message:error.message});
    }
});

router.post('/login',async (request,response)=>{
    try {
        const {username,password}=request.body;

        if(!username){
            return response.status(400).json({message:"Username Field Is Required!!!"});
        }
        if(!password){
            return response.status(400).json({message:"Password Field Is Required!!!"});
        }

        const user=await User.findUserByUsername(username);

        if(!user){
            return response.status(400).json({message:"Username Is Invalid!!!"});
        }

        const isMatch=await bcrypt.compare(password,user.password);
        if(!isMatch){
            return response.status(400).json({message:"Password Is Invalid!!!"});
        }

        const token=jwt.sign(
            {userId:user.id,name:user.name,username:user.username,email:user.email},
            'phcet',
            {expiresIn:'1hr'}
        );

        response.status(200).json({message:"Login Successfull!!!",token});
    } catch (error) {
        response.status(500).json({message:error.message});
    }
});

module.exports=router;
```

### 📝 POST /auth/register — Explanation

1. Request body se `name`, `username`, `email`, `password` nikalta hai
2. **Validation** → Har field check hoti hai, missing field par `400` error message return hota hai
3. **Duplicate check** → Username aur email already exist karte hain to `400` error return hota hai
4. **Password hashing** → `bcrypt.hash(password, 10)` se password hash hota hai (10 salt rounds). Plain password **kabhi database me store nahi hota**
5. Naya user Firestore me save hota hai
6. Success par `201 Created` status ke saath success message return hota hai

### 📝 POST /auth/login — Explanation

1. Request body se `username` aur `password` nikalta hai
2. **Validation** → Dono fields required hain
3. Username se user dhundta hai; agar nahi mila to `400` "Username Is Invalid"
4. **Password compare** → `bcrypt.compare()` user ke entered password ko database me stored hash se match karta hai
5. **JWT token generate** → `jwt.sign()` se token banta hai jisme `userId`, `name`, `username`, `email` payload hota hai:
   - Secret key: `'phcet'` (⚠️ production me isko environment variable me rakhna chahiye)
   - Expiry: `1hr` (1 ghanta)
6. Success par `200` status ke saath message + `token` return hota hai — Flutter app isi token se user details decode karti hai

---

## 🔌 API Endpoints Summary

| Method | Endpoint         | Body                                            | Response                                      |
|--------|------------------|-------------------------------------------------|-----------------------------------------------|
| POST   | `/auth/register` | `{name, username, email, password}`              | `201` → `{message: "User Created Successfully!!!"}` |
| POST   | `/auth/login`    | `{username, password}`                           | `200` → `{message: "Login Successfull!!!", token}` |

---

## 🧪 Testing (Postman / Thunder Client)

**Register:**
```
POST http://localhost:4000/auth/register
Body (JSON):
{
  "name": "John Doe",
  "username": "johndoe",
  "email": "john@example.com",
  "password": "123456"
}
```

**Login:**
```
POST http://localhost:4000/auth/login
Body (JSON):
{
  "username": "johndoe",
  "password": "123456"
}
```

---

## 🛡️ Production Tips

- JWT secret (`'phcet'`) ko `.env` file me rakho — `process.env.JWT_SECRET` use karo
- `serviceAccountKey.json` ko `.gitignore` me add karo
- `cors('*')` ki jagah sirf apne frontend ka origin allow karo
- Strong password validation (min length, special characters) add karo
