# ChatApp in Flutter

Below is a complete example of a simple Flutter chat app using Firebase Firestore for real-time messaging and Firebase Authentication for user login/signup. This example includes the UI, backend integration, and basic functionality.


### Full Code for Flutter Chat App
---
<h5> 1: Add the required dependencies: pubspec.yaml<h5>
  
  ```
    firebase_core: latest_version 
    firebase_auth: latest_version 
    cloud_firestore: latest_version
    provider: latest_version
  ```
<h5>Run flutter pub get to install the dependencies.<h5>
  
---
<h5> 2: Set up Firebase and the main app structure: main.dart<h5>
  
  ```
    import 'package:flutter/material.dart';
    import 'package:firebase_core/firebase_core.dart';
    import 'package:provider/provider.dart';
    import 'auth_service.dart';
    import 'chat_screen.dart';
    
    void main() async {
      WidgetsFlutterBinding.ensureInitialized();
      await Firebase.initializeApp();
      runApp(MyApp());
    }
    
    class MyApp extends StatelessWidget {
      @override
      Widget build(BuildContext context) {
        return MultiProvider(
          providers: [
            Provider<AuthService>(
              create: (_) => AuthService(),
            ),
          ],
          child: MaterialApp(
            title: 'Flutter Chat App',
            theme: ThemeData(
              primarySwatch: Colors.blue,
            ),
            home: AuthWrapper(),
          ),
        );
      }
    }
    
    class AuthWrapper extends StatelessWidget {
      @override
      Widget build(BuildContext context) {
        final authService = Provider.of<AuthService>(context);
        return StreamBuilder<User?>(
          stream: authService.user,
          builder: (context, snapshot) {
            if (snapshot.connectionState == ConnectionState.active) {
              final user = snapshot.data;
              return user == null ? LoginScreen() : ChatScreen();
            } else {
              return Scaffold(
                body: Center(
                  child: CircularProgressIndicator(),
                ),
              );
            }
          },
        );
      }
    }
  ```
---
<h5> 3: Handle user authentication:auth_service.dart<h5>
  
  ```
  import 'package:firebase_auth/firebase_auth.dart';

  class AuthService {
    final FirebaseAuth _auth = FirebaseAuth.instance;
  
    // Stream to listen to auth state changes
    Stream<User?> get user {
      return _auth.authStateChanges();
    }
  
    // Sign in with email and password
    Future<void> signIn(String email, String password) async {
      try {
        await _auth.signInWithEmailAndPassword(email: email, password: password);
      } catch (e) {
        print('Error signing in: $e');
      }
    }
  
    // Sign up with email and password
    Future<void> signUp(String email, String password) async {
      try {
        await _auth.createUserWithEmailAndPassword(email: email, password: password);
      } catch (e) {
        print('Error signing up: $e');
      }
    }
  
    // Sign out
    Future<void> signOut() async {
      await _auth.signOut();
    }
  }
  ```
---
<h5> 4: Create a login/signup screen: login_screen.dart<h5>
  
  ```
  import 'package:flutter/material.dart';
  import 'auth_service.dart';
  import 'package:provider/provider.dart';
  
  class LoginScreen extends StatelessWidget {
    final TextEditingController _emailController = TextEditingController();
    final TextEditingController _passwordController = TextEditingController();
  
    @override
    Widget build(BuildContext context) {
      final authService = Provider.of<AuthService>(context);
  
      return Scaffold(
        appBar: AppBar(
          title: Text('Login / Signup'),
        ),
        body: Padding(
          padding: const EdgeInsets.all(16.0),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              TextField(
                controller: _emailController,
                decoration: InputDecoration(labelText: 'Email'),
              ),
              TextField(
                controller: _passwordController,
                decoration: InputDecoration(labelText: 'Password'),
                obscureText: true,
              ),
              SizedBox(height: 20),
              ElevatedButton(
                onPressed: () async {
                  await authService.signIn(
                    _emailController.text,
                    _passwordController.text,
                  );
                },
                child: Text('Login'),
              ),
              ElevatedButton(
                onPressed: () async {
                  await authService.signUp(
                    _emailController.text,
                    _passwordController.text,
                  );
                },
                child: Text('Signup'),
              ),
            ],
          ),
        ),
      );
    }
  }
  ```
---
<h5> 5: Create the chat screen with real-time messaging: chat_screen.dart <h5>
   
```
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter/material.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'auth_service.dart';
import 'package:provider/provider.dart';

class ChatScreen extends StatelessWidget {
  final TextEditingController _messageController = TextEditingController();
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  ChatScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final authService = Provider.of<AuthService>(context);
    final userStream = authService.user;

    return Scaffold(
      appBar: AppBar(
        title: const Text('Chat App'),
        actions: [
          IconButton(
            icon: const Icon(Icons.logout),
            onPressed: () async {
              await authService.signOut();
            },
          ),
        ],
      ),
      body: Column(
        children: [
          Expanded(
            child: StreamBuilder<QuerySnapshot>(
              stream: _firestore
                  .collection('messages')
                  .orderBy('timestamp', descending: false)
                  .snapshots(),
              builder: (context, snapshot) {
                if (!snapshot.hasData) {
                  return const Center(
                    child: CircularProgressIndicator(),
                  );
                }

                final messages = snapshot.data!.docs;
                List<Widget> messageWidgets = [];
                for (var message in messages) {
                  final messageText = message['text'];
                  final messageSender = message['sender'];

                  final messageWidget = ListTile(
                    title: Text(messageText),
                    subtitle: Text(messageSender),
                  );
                  messageWidgets.add(messageWidget);
                }

                return ListView(
                  children: messageWidgets,
                );
              },
            ),
          ),
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: Row(
              children: [
                Expanded(
                  child: TextField(
                    controller: _messageController,
                    decoration: const InputDecoration(
                      hintText: 'Enter your message...',
                    ),
                  ),
                ),
                StreamBuilder<User?>(
                  stream: userStream,
                  builder: (context, snapshot) {
                    if (!snapshot.hasData) {
                      return const IconButton(
                        icon: Icon(Icons.send),
                        onPressed: null,
                      );
                    }
                    final user = snapshot.data;
                    return IconButton(
                      icon: const Icon(Icons.send),
                      onPressed: () async {
                        if (_messageController.text.isNotEmpty) {
                          await _firestore.collection('messages').add({
                            'text': _messageController.text,
                            'sender': user!.email,
                            'timestamp': FieldValue.serverTimestamp(),
                          });
                          _messageController.clear();
                        }
                      },
                    );
                  },
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```
---

## Connect

<a href="https://dev-aryanbhimani.pantheonsite.io/" target="_blank"><img src="portfolio.png" width="50" ></a>
<a href="https://www.linkedin.com/in/aryanbhimani/" target="_blank"><img src="linkedin.png" width="50"></a>
<a href="https://x.com/aryan46022" target="_blank"><img src="twitter.png" width="50"></a> 

For queries or support, feel free to reach out:  
📞 **+91 9408962204**  
📧 **aryan.bhimani.93@email.com**

---
