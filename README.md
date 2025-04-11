import 'dart:async';
import 'dart:io';

import 'package:camera/camera.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_tflite/flutter_tflite.dart';
import 'package:flutter_tts/flutter_tts.dart';

import 'image.dart';

void main() {
  runApp(const Home());
}

class Home extends StatelessWidget {
  const Home({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'currency and tablet detection',
      theme: ThemeData(
        primarySwatch: Colors.deepPurple,
        visualDensity: VisualDensity.adaptivePlatformDensity,
      ),
      home: const CameraScreen(),
    );
  }
}

class CameraScreen extends StatefulWidget {
  const CameraScreen({Key? key}) : super(key: key);

  @override
  State<CameraScreen> createState() => _CameraScreenState();
}

class _CameraScreenState extends State<CameraScreen> {
  CameraController? _controller;
  XFile? _imageFile;
  bool _isProcessing = false;
  final FlutterTts _flutterTts = FlutterTts();

  @override
  void initState() {
    super.initState();
    _configureTts();
    _initializeCamera().then((_) {
      _flutterTts.speak('Your camera is now open. Please position your tablet correctly.').then((_) {
        Timer(const Duration(seconds: 10), _captureAndClassifyImage);
      });
    });
    _tfLiteInit();
  }

  void _configureTts() {
    _flutterTts.setLanguage("en-US");
    _flutterTts.setSpeechRate(0.3);
    _flutterTts.setVolume(1.0);
    _flutterTts.setPitch(1.0);
  }

  Future<void> _initializeCamera() async {
    final cameras = await availableCameras();
    final firstCamera = cameras.first;

    _controller = CameraController(
      firstCamera,
      ResolutionPreset.high,
      enableAudio: false,
    );

    try {
      await _controller!.initialize();
      if (!mounted) return;
      setState(() {});
    } catch (e) {
      print("Error initializing camera: $e");
    }
  }

  Future<void> _tfLiteInit() async {
    try {
      final res = await Tflite.loadModel(
        model: "assets/model.tflite",
        labels: "assets/label_money.txt",
        numThreads: 1,
        isAsset: true,
        useGpuDelegate: false,
      );
      print('Model loaded: $res');
    } on PlatformException catch (e) {
      print('Failed to load model: $e');
    }
  }

  Future<void> _captureAndClassifyImage() async {
    if (_controller == null || !_controller!.value.isInitialized || _isProcessing) {
      print("Controller is not initialized or already processing");
      return;
    }

    setState(() {
      _isProcessing = true;
    });

    try {
      print("AutoCapture: Taking picture...");
      final XFile imageFile = await _controller!.takePicture();
      setState(() {
        _imageFile = imageFile;
      });

      // Close the camera after capturing the image
      await _controller!.dispose();
      _controller = null;

      // Navigate to ImageProcessingScreen with the captured image
      Navigator.push(
        context,
        MaterialPageRoute(
          builder: (context) => ImageProcessingScreen(
            imagePath: imageFile.path,
          ),
        ),
      );
    } catch (e) {
      print("Error taking picture: $e");
    } finally {
      setState(() {
        _isProcessing = false;
      });
    }
  }

  @override
  void dispose() {
    _controller?.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Visionary touch'),
      ),
      body: Stack(
        children: [
          if (_controller?.value.isInitialized ?? false)
            CameraPreview(_controller!)
          else
            const Center(child: CircularProgressIndicator()),
          if (_imageFile != null)
            Positioned(
              bottom: 0,
              left: 0,
              right: 0,
              child: Container(
                color: Colors.black54,
                padding: const EdgeInsets.all(16.0),
                child: Image.file(
                  File(_imageFile!.path),
                  height: 200,
                  width: double.infinity,
                  fit: BoxFit.cover,
                ),
              ),
            ),
        ],
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _captureAndClassifyImage,
        child: const Icon(Icons.camera_alt),
      ),
    );
  }
}
