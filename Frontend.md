# 📱 Auth App Webapp (Flutter Frontend)

Flutter se bana ek **Authentication App** jo `auth-app-api` (Node.js backend) se connect hota hai. Isme **Register**, **Login** aur **Profile** screens hain. Login ke baad JWT token decode karke user details profile par show hoti hain.

---

## 📁 Project Structure (lib folder)

```
lib/
├── main.dart                    # App entry point + routes
├── models/
│   └── user.dart                # User data model
├── screens/
│   ├── home.dart                # Home screen (Register/Login buttons)
│   ├── login.dart               # Login screen
│   ├── register.dart            # Register screen
│   └── profile.dart             # Profile screen (user details)
├── services/
│   └── user_service.dart        # API calls (backend se communication)
└── utils/
    └── custom_alert_box.dart    # Custom styled alert dialogs
```

---

## 📦 Dependencies (pubspec.yaml)

```yaml
dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.8
  http: ^1.1.0
  jwt_decode: ^0.3.1
```

**Explanation:**
- **http** → Backend API calls karne ke liye (register/login requests)
- **jwt_decode** → JWT token ko decode karne ke liye (login ke baad user details nikalne ke liye)
- **cupertino_icons** → iOS style icons ke liye

**Install karne ke liye:**
```bash
flutter pub get
```

---

## 🚀 lib/main.dart (App Entry Point + Routes)

```dart
import 'package:auth_app_webapp/screens/home.dart';
import 'package:auth_app_webapp/screens/login.dart';
import 'package:auth_app_webapp/screens/profile.dart';
import 'package:auth_app_webapp/screens/register.dart';
import 'package:flutter/material.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  Widget build(BuildContext context) {
    return MaterialApp(
      title: "Auth App",
      initialRoute: '/',
      routes: {
        '/': (context) => HomeScreen(),
        '/register': (context) => RegisterScreen(),
        '/login': (context) => LoginScreen(),
        '/profile': (context) => ProfileScreen(),
      },
    );
  }
}
```

**Explanation:**
1. `main()` → App ka starting point, `runApp()` se `MyApp` widget run hota hai
2. `MaterialApp` → App ki root widget, theme/navigation handle karti hai
3. `initialRoute: '/'` → App start hone par sabse pehle `HomeScreen` khulta hai
4. `routes` map → Named routing define karta hai. Kahi bhi `Navigator.pushNamed(context, '/register')` likhoge to `RegisterScreen` khul jayega. Isse navigation clean aur reusable rehti hai
5. Ye 4 routes hain: `/` (home), `/register`, `/login`, `/profile`

---

## 👤 lib/models/user.dart (User Model)

```dart
class User {
  String id;
  String name;
  String username;
  String email;
  String password;

  User({
    required this.id,
    required this.name,
    required this.username,
    required this.email,
    required this.password,
  });

  factory User.fromJson(Map<String, dynamic> map) {
    return User(
      id: map['id'],
      name: map['name'],
      username: map['username'],
      email: map['email'],
      password: map['password'],
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'name': name,
      'username': username,
      'email': email,
      'password': password,
    };
  }
}
```

**Explanation:**
- Simple data class jo user ki information hold karti hai (`id`, `name`, `username`, `email`, `password`)
- Constructor me `required` keyword hai — matlab ye saare fields dena mandatory hai
- **`User.fromJson()`** → Factory constructor jo Map (JSON) se `User` object banata hai — API response aane par use hota hai
- **`toJson()`** → `User` object ko Map me convert karta hai — API ko **register request** bhejne ke time `jsonEncode(user.toJson())` se JSON body banane ke liye use hota hai
- Register screen me `id: ""` empty pass hota hai kyunki backend Firestore me document banate waqt khud unique id generate karta hai

---

## 🏠 lib/screens/home.dart (Home Screen)

```dart
import 'package:flutter/material.dart';

class HomeScreen extends StatelessWidget {
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("Home Screen")),
      body: Center(
        child: Column(
          children: [
            TextButton(
              onPressed: () => {Navigator.pushNamed(context, '/register')},
              child: Text("Go To Register Page"),
            ),
            TextButton(
              onPressed: () => {Navigator.pushNamed(context, '/login')},
              child: Text("Go To Login Page"),
            ),
          ],
        ),
      ),
    );
  }
}
```

**Explanation:**
- `StatelessWidget` hai kyunki is screen ka koi changing state nahi hai
- `Scaffold` → Screen ka basic layout structure (appBar + body) deta hai
- Do `TextButton` hain jo `Navigator.pushNamed()` se named routes par navigate karte hain (`/register` aur `/login`)
- Buttons center me hain `Center` + `Column` widget se

---

## 🔐 lib/screens/login.dart (Login Screen)

```dart
import 'package:auth_app_webapp/services/user_service.dart';
import 'package:auth_app_webapp/utils/custom_alert_box.dart';
import 'package:flutter/material.dart';
import 'package:jwt_decode/jwt_decode.dart';
import '../models/user.dart';
import 'dart:async';

class LoginScreen extends StatefulWidget {
  LoginScreenState createState() => LoginScreenState();
}

class LoginScreenState extends State<LoginScreen> {
  TextEditingController usernameController = TextEditingController();
  TextEditingController passwordController = TextEditingController();

  void handleSubmit() async {
    final response = await UserService.login(
      usernameController.text,
      passwordController.text,
    );
    String message = response['message'];
    if (message == "Login Successfull!!!") {
      final token = response['token'];
      Map<String, dynamic> decodedToken = Jwt.parseJwt(token);
      User user = User(
        id: decodedToken['userId'],
        name: decodedToken['name'],
        username: usernameController.text,
        email: decodedToken['email'],
        password: "",
      );
      CustomAlertBox.showSuccess(context, "Success", message);
      Timer(Duration(seconds: 2), () {
        Navigator.pushNamed(context, '/profile', arguments: user);
      });
    } else {
      CustomAlertBox.showError(context, "Error", message);
    }
  }

  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("Login Screen")),
      body: Column(
        children: [
          TextField(
            controller: usernameController,
            decoration: InputDecoration(labelText: "Username"),
          ),

          TextField(
            controller: passwordController,
            decoration: InputDecoration(labelText: "Password"),
          ),
          TextButton(onPressed: handleSubmit, child: Text("Login")),
        ],
      ),
    );
  }
}
```

**Explanation:**
1. `StatefulWidget` hai kyunki text fields me user type karta hai (changing state)
2. **TextEditingControllers** → `usernameController` aur `passwordController` text fields ka content read karne ke liye. `.text` property se current value milti hai
3. **`handleSubmit()`** (Login button press hone par):
   - `UserService.login()` se backend par login request jaati hai
   - Response ka `message` check hota hai — agar `"Login Successfull!!!"` hai to:
     - Token milta hai response se
     - `Jwt.parseJwt(token)` se token decode karke `userId`, `name`, `email` nikalte hain
     - `User` object banta hai aur `CustomAlertBox.showSuccess()` se success dialog dikhta hai
     - `Timer` 2 second baad `/profile` route par navigate karta hai, aur `arguments: user` se `User` object profile screen ko pass hota hai
   - Agar message match nahi hua (error) → `CustomAlertBox.showError()` se error dialog dikhta hai

---

## 📝 lib/screens/register.dart (Register Screen)

```dart
import 'package:auth_app_webapp/services/user_service.dart';
import 'package:auth_app_webapp/utils/custom_alert_box.dart';
import 'package:flutter/material.dart';
import '../models/user.dart';
import 'dart:async';

class RegisterScreen extends StatefulWidget {
  RegisterScreenState createState() => RegisterScreenState();
}

class RegisterScreenState extends State<RegisterScreen> {
  TextEditingController nameController = TextEditingController();
  TextEditingController usernameController = TextEditingController();
  TextEditingController emailController = TextEditingController();
  TextEditingController passwordController = TextEditingController();

  void handleSubmit() async {
    User user = User(
      id: "",
      name: nameController.text,
      username: usernameController.text,
      email: emailController.text,
      password: passwordController.text,
    );
    final response = await UserService.register(user);
    String message = response['message'];
    if (message == "User Created Successfully!!!") {
      CustomAlertBox.showSuccess(context, "Success", message);
      Timer(Duration(seconds: 2), () {
        Navigator.pushNamed(context, '/login');
      });
    } else {
      CustomAlertBox.showError(context, "Error", message);
    }
  }

  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("Register Screen")),
      body: Column(
        children: [
          TextField(
            controller: nameController,
            decoration: InputDecoration(labelText: "Name"),
          ),
          TextField(
            controller: usernameController,
            decoration: InputDecoration(labelText: "Username"),
          ),
          TextField(
            controller: emailController,
            decoration: InputDecoration(labelText: "Email"),
          ),
          TextField(
            controller: passwordController,
            decoration: InputDecoration(labelText: "Password"),
          ),
          TextButton(onPressed: handleSubmit, child: Text("Register")),
        ],
      ),
    );
  }
}
```

**Explanation:**
1. `StatefulWidget` hai — 4 text fields (Name, Username, Email, Password) ke controllers hain
2. **`handleSubmit()`** (Register button press hone par):
   - Text fields se values lekar ek `User` object banta hai (`id: ""` empty pass hota hai — backend khud id generate karta hai)
   - `UserService.register(user)` se backend par register request jaati hai
   - Response ka `message` check hota hai:
     - `"User Created Successfully!!!"` → success dialog + 2 second baad login screen par navigate
     - Warna (duplicate username/email ya validation error) → error dialog

---

## 👤 lib/screens/profile.dart (Profile Screen)

```dart
import 'package:flutter/material.dart';
import '../models/user.dart';

class ProfileScreen extends StatelessWidget {
  Widget build(BuildContext context) {
    User user = ModalRoute.of(context)!.settings.arguments as User;
    return Scaffold(
      appBar: AppBar(title: Text("Profile Screen")),
      body: Center(
        child: Column(
          children: [
            Text('Id :- ${user.id}'),
            Text('Name :- ${user.name}'),
            Text('Username :- ${user.username}'),
            Text('Email :- ${user.email}'),
            TextButton(
              onPressed: () {
                Navigator.pushNamed(context, '/');
              },
              child: Text("Go To Home Page"),
            ),
          ],
        ),
      ),
    );
  }
}
```

**Explanation:**
1. `StatelessWidget` hai — sirf data show karna hai, koi interaction nahi
2. **`ModalRoute.of(context)!.settings.arguments as User`** → Login screen se bheja gaya `User` object yahan receive hota hai. Login screen me `arguments: user` se jo pass kiya tha, wahi yahan milta hai
3. `user.id`, `user.name`, `user.username`, `user.email` — user ki details `Text` widgets me show hoti hain
4. "Go To Home Page" button home screen par wapas le jaata hai

---

## 🌐 lib/services/user_service.dart (API Service)

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import '../models/user.dart';

class UserService {
  static String API_URL = "http://localhost:4000/auth";

  static Future<Map<String, dynamic>> register(User user) async {
    final response = await http.post(
      Uri.parse('$API_URL/register'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode(user.toJson()),
    );
    return jsonDecode(response.body);
  }

  static Future<Map<String, dynamic>> login(
    String username,
    String password,
  ) async {
    final response = await http.post(
      Uri.parse('$API_URL/login'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({'username': username, 'password': password}),
    );
    return jsonDecode(response.body);
  }
}
```

**Explanation:**
- Ye class poori **backend communication** handle karti hai. `API_URL` me backend ka base URL hai (`http://localhost:4000/auth` — local development ke liye)
- **`register(User user)`** → `http.post()` se `POST /auth/register` request jaati hai. `user.toJson()` ko `jsonEncode()` se JSON string banaya gaya hai, aur `Content-Type: application/json` header set kiya gaya hai. Response ka body `jsonDecode()` se Map me convert karke return hota hai
- **`login(username, password)`** → `POST /auth/login` par username/password JSON body ke saath bhejta hai aur response return karta hai
- Dono methods `static` hain — bina object banaye direct `UserService.login(...)` call kar sakte hain
- `Future<Map<String, dynamic>>` → async response ka matlab hai; `await` se result milta hai

⚠️ **Note:** Android emulator par `localhost` kaam nahi karega — wahan `http://10.0.2.2:4000/auth` use karna hoga (10.0.2.2 emulator ke liye host machine ka localhost hota hai). Real device par apne PC ka IP use karo.

---

## 🔔 lib/utils/custom_alert_box.dart (Custom Alert Dialogs)

```dart
import 'package:flutter/material.dart';

class CustomAlertBox {
  static void showSuccess(BuildContext context, String title, String message) {
    _showAlert(
      context: context,
      title: title,
      message: message,
      icon: Icons.check_circle,
      iconColor: Colors.green,
      backgroundColor: Colors.green.shade50,
      titleColor: Colors.green.shade800,
      messageColor: Colors.green.shade700,
    );
  }

  static void showError(BuildContext context, String title, String message) {
    _showAlert(
      context: context,
      title: title,
      message: message,
      icon: Icons.error,
      iconColor: Colors.red,
      backgroundColor: Colors.red.shade50,
      titleColor: Colors.red.shade800,
      messageColor: Colors.red.shade700,
    );
  }

  static void showWarning(BuildContext context, String title, String message) {
    _showAlert(
      context: context,
      title: title,
      message: message,
      icon: Icons.warning,
      iconColor: Colors.orange,
      backgroundColor: Colors.orange.shade50,
      titleColor: Colors.orange.shade800,
      messageColor: Colors.orange.shade700,
    );
  }

  static void showInfo(BuildContext context, String title, String message) {
    _showAlert(
      context: context,
      title: title,
      message: message,
      icon: Icons.info,
      iconColor: Colors.blue,
      backgroundColor: Colors.blue.shade50,
      titleColor: Colors.blue.shade800,
      messageColor: Colors.blue.shade700,
    );
  }

  static void _showAlert({
    required BuildContext context,
    required String title,
    required String message,
    required IconData icon,
    required Color iconColor,
    required Color backgroundColor,
    required Color titleColor,
    required Color messageColor,
  }) {
    showDialog(
      context: context,
      barrierDismissible: true,
      builder: (BuildContext context) {
        return AlertDialog(
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(15),
          ),
          backgroundColor: backgroundColor,
          contentPadding: const EdgeInsets.all(20),
          content: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Icon(icon, size: 50, color: iconColor),
              const SizedBox(height: 15),
              Text(
                title,
                style: TextStyle(
                  fontSize: 20,
                  fontWeight: FontWeight.bold,
                  color: titleColor,
                ),
                textAlign: TextAlign.center,
              ),
              const SizedBox(height: 10),
              Text(
                message,
                style: TextStyle(fontSize: 16, color: messageColor),
                textAlign: TextAlign.center,
              ),
              const SizedBox(height: 20),
              ElevatedButton(
                onPressed: () {
                  Navigator.of(context).pop();
                },
                style: ElevatedButton.styleFrom(
                  backgroundColor: iconColor,
                  foregroundColor: Colors.white,
                  shape: RoundedRectangleBorder(
                    borderRadius: BorderRadius.circular(10),
                  ),
                  padding: const EdgeInsets.symmetric(
                    horizontal: 25,
                    vertical: 12,
                  ),
                ),
                child: const Text('OK'),
              ),
            ],
          ),
        );
      },
    );
  }
}
```

**Explanation:**
- Ye ek reusable utility class hai jo **4 types ke styled dialogs** dikhata hai:
  - `showSuccess()` → 🟢 Green theme (check icon) — successful operations ke liye
  - `showError()` → 🔴 Red theme (error icon) — errors ke liye
  - `showWarning()` → 🟠 Orange theme (warning icon) — warnings ke liye
  - `showInfo()` → 🔵 Blue theme (info icon) — general information ke liye
- Sabhi methods andar `_showAlert()` private method ko call karte hain, sirf colors/icons different hote hain — **code duplication se bachav** hota hai
- `_showAlert()` me:
  - `showDialog()` screen ke upar dialog dikhata hai
  - `barrierDismissible: true` → dialog ke bahar tap karne se band ho jaata hai
  - `AlertDialog` me icon, title, message aur ek **OK button** hai jo `Navigator.of(context).pop()` se dialog band karta hai
  - `mainAxisSize: MainAxisSize.min` → Column sirf utni hi jagah leta hai jitna content hai

---

## 🔄 App Flow (Kaise Kaam Karta Hai)

```
Home Screen (/)
    │
    ├── "Go To Register Page" ──→ Register Screen (/register)
    │                                  │  (form fill karke Register dabao)
    │                                  │  Success → Login Screen
    │
    └── "Go To Login Page" ──→ Login Screen (/login)
                                   │  (username/password daalo)
                                   │  Success → JWT decode → Profile Screen (/profile)
                                   │              User object arguments se pass hota hai
                                   │
                                   └── Profile Screen → user details show + Home par wapas
```

---

## ▶️ Run Kaise Kare

1. Pehle backend start karo (`auth-app-api` folder me): `node server.js`
2. Phir Flutter app chalao:
```bash
flutter pub get
flutter run
```
3. Emulator/Device par app khulegi → Register karo → Login karo → Profile dekho ✅

