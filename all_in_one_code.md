# Face Swapper Pro (All-in-One Repo)

Bu dosya projedeki tüm dosyaları içerir.  
Codemagic veya setup script, buradan klasör yapısını oluşturabilir.  

---

## FILE: mobile_app/lib/main.dart
```dart
import 'package:flutter/material.dart';
import 'package:local_auth/local_auth.dart';
import 'package:file_picker/file_picker.dart';
import 'dart:async';
import 'dart:io';
import 'package:flutter/services.dart';
import 'package:http/http.dart' as http;
import 'package:path_provider/path_provider.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await downloadModels(); // 🔥 ilk açılışta modelleri indir
  runApp(FaceSwapperApp());
}

Future<void> downloadModels() async {
  final dir = await getApplicationDocumentsDirectory();
  final modelDir = Directory("${dir.path}/models");
  if (!modelDir.existsSync()) {
    modelDir.createSync(recursive: true);
  }

  final models = {
    "InsightFace.onnx": "https://huggingface.co/deepinsight/insightface/resolve/main/models/inswapper_128.onnx",
    "YOLOv8-Face.onnx": "https://huggingface.co/onnx/models/resolve/main/face-detection/yolov8n-face.onnx",
  };

  for (var entry in models.entries) {
    final filePath = "${modelDir.path}/${entry.key}";
    if (!File(filePath).existsSync()) {
      print("Downloading ${entry.key}...");
      final response = await http.get(Uri.parse(entry.value));
      if (response.statusCode == 200) {
        await File(filePath).writeAsBytes(response.bodyBytes);
        print("${entry.key} indirildi!");
      } else {
        throw Exception("Model indirilemedi: ${entry.key}");
      }
    }
  }
}

class FaceSwapperApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Face Swapper Pro',
      debugShowCheckedModeBanner: false,
      theme: ThemeData.dark(),
      home: AuthGate(),
    );
  }
}

class AuthGate extends StatefulWidget {
  @override
  _AuthGateState createState() => _AuthGateState();
}

class _AuthGateState extends State<AuthGate> {
  final LocalAuthentication auth = LocalAuthentication();

  Future<void> _authenticate() async {
    try {
      bool didAuth = await auth.authenticate(
        localizedReason: 'Yalnızca senin erişimin için FaceID/Parmak izi doğrulaması',
        options: const AuthenticationOptions(biometricOnly: true),
      );
      if (didAuth) {
        Navigator.pushReplacement(
            context, MaterialPageRoute(builder: (context) => HomePage()));
      }
    } catch (e) {
      print("Auth error: $e");
    }
  }

  @override
  void initState() {
    super.initState();
    _authenticate();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
        body: Center(child: CircularProgressIndicator()));
  }
}

class HomePage extends StatefulWidget {
  @override
  _HomePageState createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  String? _videoPath;
  static const platform = MethodChannel("face_swap_channel");

  Future<void> _pickVideo() async {
    final result = await FilePicker.platform.pickFiles(type: FileType.video);
    if (result != null) {
      setState(() {
        _videoPath = result.files.single.path!;
      });
    }
  }

  Future<void> _startSwap() async {
    if (_videoPath == null) return;

    try {
      final outputPath = await platform.invokeMethod("runFaceSwap", {
        "inputPath": _videoPath!,
      });

      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text("Face swap tamamlandı: $outputPath")),
      );
    } on PlatformException catch (e) {
      print("Face swap hata: ${e.message}");
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("Face Swapper Pro")),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(onPressed: _pickVideo, child: Text("Video Seç")),
            if (_videoPath != null)
              Padding(
                padding: const EdgeInsets.all(12.0),
                child: Text(
                  "Seçilen video: $_videoPath",
                  textAlign: TextAlign.center,
                ),
              ),
            if (_videoPath != null)
              ElevatedButton(
                onPressed: _startSwap,
                child: Text("Face Swap Başlat"),
              ),
          ],
        ),
      ),
    );
  }
}
