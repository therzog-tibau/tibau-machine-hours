# tibau-machine-hours
// =============================
// pubspec.yaml (in Projektwurzel)
// =============================
/*
name: tibau_machine_hours
description: MVP zum Testen von QR-Codes, GPS, Stunden und OCR
publish_to: 'none'
version: 0.1.0+1

environment:
  sdk: '>=3.3.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.6
  mobile_scanner: ^6.0.2
  google_mlkit_text_recognition: ^0.14.0
  geolocator: ^13.0.1
  image_picker: ^1.1.2
  path_provider: ^2.1.4
  hive: ^2.2.3
  hive_flutter: ^1.1.0
  intl: ^0.19.0

flutter:
  uses-material-design: true
*/

// =============================
// lib/main.dart
// =============================
import 'dart:convert';
import 'dart:io';

import 'package:flutter/material.dart';
import 'package:geolocator/geolocator.dart';
import 'package:google_mlkit_text_recognition/google_mlkit_text_recognition.dart';
import 'package:hive/hive.dart';
import 'package:hive_flutter/hive_flutter.dart';
import 'package:image_picker/image_picker.dart';
import 'package:intl/intl.dart';
import 'package:mobile_scanner/mobile_scanner.dart';
import 'package:path_provider/path_provider.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final Directory appDocDir = await getApplicationDocumentsDirectory();
  await Hive.initFlutter(appDocDir.path);
  Hive.registerAdapter(ReadingAdapter());
  await Hive.openBox<Reading>('readings');
  runApp(const App());
}

class App extends StatelessWidget {
  const App({super.key});
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Tibau Stundenerfassung',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blueGrey),
        useMaterial3: true,
      ),
      home: const HomePage(),
    );
  }
}

class HomePage extends StatefulWidget {
  const HomePage({super.key});
  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  Map<String, dynamic>? scannedMachine; // {id, name, serviceintervall_h}
  Position? position;
  final _hoursCtrl = TextEditingController();
  File? photoFile;
  bool saving = false;

  @override
  void dispose() {
    _hoursCtrl.dispose();
    super.dispose();
  }

  Future<void> _scanQr() async {
    await Navigator.of(context).push(
      MaterialPageRoute(
        builder: (_) => const QrScanPage(),
      ),
    ).then((value) {
      if (value is String) {
        try {
          final Map<String, dynamic> data = json.decode(value);
          if (data['id'] != null && data['name'] != null) {
            setState(() => scannedMachine = data);
          } else {
            _showSnack('QR ungültig: Felder fehlen');
          }
        } catch (_) {
          _showSnack('QR ist kein gültiges JSON');
        }
      }
    });
  }

  Future<void> _getGps() async {
    bool serviceEnabled = await Geolocator.isLocationServiceEnabled();
    if (!serviceEnabled) {
      _showSnack('GPS ist deaktiviert. Bitte aktivieren.');
      return;
    }
    LocationPermission permission = await Geolocator.checkPermission();
    if (permission == LocationPermission.denied) {
      permission = await Geolocator.requestPermission();
      if (permission == LocationPermission.denied) {
        _showSnack('GPS-Berechtigung verweigert.');
        return;
      }
    }
    if (permission == LocationPermission.deniedForever) {
      _showSnack('GPS dauerhaft verweigert. In Einstellungen erlauben.');
      return;
    }
    final pos = await Geolocator.getCurrentPosition(desiredAccuracy: LocationAccuracy.best);
    setState(() => position = pos);
  }

  Future<void> _pickPhotoAndOcr() async {
    final ImagePicker picker = ImagePicker();
    final XFile? photo = await picker.pickImage(source: ImageSource.camera, imageQuality: 85);
    if (photo == null) return;
    setState(() => photoFile = File(photo.path));

    final inputImage = InputImage.fromFilePath(photo.path);
    final textRecognizer = TextRecognizer(script: TextRecognitionScript.latin);
    final RecognizedText recognizedText = await textRecognizer.processImage(inputImage);
    await textRecognizer.close();

    // einfache Filterung: nimm die längste reine Ziffern-/Kommazahl
    String? best;
    int bestLen = 0;
    final reg = RegExp(r'^[0-9]{1,7}([\.,][0-9])?$');
    for (final block in recognizedText.blocks) {
      for (final line in block.lines) {
        final t = line.text.replaceAll(' ', '');
        if (reg.hasMatch(t) && t.length > bestLen) {
          best = t;
          bestLen = t.length;
        }
      }
    }
    if (best != null) {
      // Komma zu Punkt
      best = best!.replaceAll(',', '.');
      _hoursCtrl.text = best!;
      _showSnack('OCR erkannt: $best');
    } else {
      _showSnack('Keine Stunden erkannt. Bitte manuell eintragen.');
    }
  }

  Future<void> _saveReading() async {
    if (scannedMachine == null) {
      _showSnack('Bitte zuerst QR scannen.');
      return;
    }
    if (_hoursCtrl.text.trim().isEmpty) {
      _showSnack('Bitte Stunden eingeben oder via OCR erfassen.');
      return;
    }
    final h = double.tryParse(_hoursCtrl.text.replaceAll(',', '.'));
    if (h == null) {
      _showSnack('Stundenwert ungültig.');
      return;
    }
    setState(() => saving = true);

    final box = Hive.box<Reading>('readings');
    final r = Reading(
      machineId: scannedMachine!['id'],
      machineName: scannedMachine!['name'],
      serviceIntervalH: (scannedMachine!['serviceintervall_h'] is int)
          ? scannedMachine!['serviceintervall_h']
          : int.tryParse('${scannedMachine!['serviceintervall_h'] ?? ''}') ?? 250,
      hours: h,
      photoPath: photoFile?.path,
      lat: position?.latitude,
      lng: position?.longitude,
      createdAt: DateTime.now(),
    );
    await box.add(r);

    setState(() {
      saving = false;
      _hoursCtrl.clear();
      photoFile = null;
    });
    _showSnack('Erfassung gespeichert.');
  }

  @override
  Widget build(BuildContext context) {
    final df = DateFormat('dd.MM.yyyy HH:mm');
    return Scaffold(
      appBar: AppBar(title: const Text('Stundenerfassung (MVP)')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          Row(
            children: [
              Expanded(
                child: OutlinedButton.icon(
                  onPressed: _scanQr,
                  icon: const Icon(Icons.qr_code_scanner),
                  label: const Text('QR scannen'),
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: OutlinedButton.icon(
                  onPressed: _getGps,
                  icon: const Icon(Icons.my_location),
                  label: const Text('GPS erfassen'),
                ),
              ),
            ],
          ),
          const SizedBox(height: 12),
          if (scannedMachine != null)
            Card(
              child: Padding(
                padding: const EdgeInsets.all(12.0),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text('Gerät', style: Theme.of(context).textTheme.titleMedium),
                    Text('Nr.: ${scannedMachine!['id']}'),
                    Text('Name: ${scannedMachine!['name']}'),
                    if (scannedMachine!['serviceintervall_h'] != null)
                      Text('Serviceintervall: ${scannedMachine!['serviceintervall_h']} h'),
                  ],
                ),
              ),
            ),
          const SizedBox(height: 8),
          TextField(
            controller: _hoursCtrl,
            keyboardType: const TextInputType.numberWithOptions(decimal: true),
            decoration: const InputDecoration(
              labelText: 'Stundenstand',
              hintText: 'z. B. 1280.5',
              border: OutlineInputBorder(),
            ),
          ),
          const SizedBox(height: 8),
          Row(
            children: [
              Expanded(
                child: OutlinedButton.icon(
                  onPressed: _pickPhotoAndOcr,
                  icon: const Icon(Icons.photo_camera),
                  label: const Text('Foto + OCR'),
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: ElevatedButton.icon(
                  onPressed: saving ? null : _saveReading,
                  icon: const Icon(Icons.save),
                  label: Text(saving ? 'Speichere…' : 'Speichern'),
                ),
              ),
            ],
          ),
          const SizedBox(height: 16),
          if (position != null)
            Text('Standort: ${position!.latitude.toStringAsFixed(6)}, ${position!.longitude.toStringAsFixed(6)}'),
          const SizedBox(height: 16),
          const Divider(),
          Text('Lokale Liste', style: Theme.of(context).textTheme.titleMedium),
          const SizedBox(height: 8),
          ValueListenableBuilder(
            valueListenable: Hive.box<Reading>('readings').listenable(),
            builder: (context, Box<Reading> box, _) {
              final items = box.values.toList().reversed.toList();
              if (items.isEmpty) {
                return const Text('Noch keine Einträge.');
              }
              return Column(
                children: [
                  for (final r in items)
                    Card(
                      child: ListTile(
                        title: Text('${r.machineId}  •  ${r.hours.toStringAsFixed(1)} h'),
                        subtitle: Text(
                          '${r.machineName}\n${df.format(r.createdAt)}\n${r.lat != null ? 'GPS: ${r.lat!.toStringAsFixed(6)}, ${r.lng!.toStringAsFixed(6)}' : ''}',
                        ),
                        trailing: r.photoPath != null
                            ? const Icon(Icons.photo, size: 20)
                            : const SizedBox.shrink(),
                        onTap: () {
                          if (r.photoPath != null) {
                            Navigator.of(context).push(
                              MaterialPageRoute(
                                builder: (_) => PhotoPage(path: r.photoPath!),
                              ),
                            );
                          }
                        },
                      ),
                    ),
                ],
              );
            },
          ),
        ],
      ),
    );
  }

  void _showSnack(String msg) {
    ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(msg)));
  }
}

class QrScanPage extends StatelessWidget {
  const QrScanPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('QR scannen')),
      body: MobileScanner(
        onDetect: (capture) {
          final barcodes = capture.barcodes;
          for (final barcode in barcodes) {
            final raw = barcode.rawValue;
            if (raw != null && raw.isNotEmpty) {
              Navigator.of(context).pop(raw);
              break;
            }
          }
        },
      ),
    );
  }
}

class PhotoPage extends StatelessWidget {
  final String path;
  const PhotoPage({super.key, required this.path});
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Foto')),
      body: Center(child: Image.file(File(path))),
    );
  }
}

@HiveType(typeId: 1)
class Reading extends HiveObject {
  @HiveField(0)
  String machineId;
  @HiveField(1)
  String machineName;
  @HiveField(2)
  int serviceIntervalH;
  @HiveField(3)
  double hours;
  @HiveField(4)
  String? photoPath;
  @HiveField(5)
  double? lat;
  @HiveField(6)
  double? lng;
  @HiveField(7)
  DateTime createdAt;

  Reading({
    required this.machineId,
    required this.machineName,
    required this.serviceIntervalH,
    required this.hours,
    this.photoPath,
    this.lat,
    this.lng,
    required this.createdAt,
  });
}

class ReadingAdapter extends TypeAdapter<Reading> {
  @override
  final int typeId = 1;

  @override
  Reading read(BinaryReader reader) {
    return Reading(
      machineId: reader.readString(),
      machineName: reader.readString(),
      serviceIntervalH: reader.readInt(),
      hours: reader.readDouble(),
      photoPath: reader.readBool() ? reader.readString() : null,
      lat: reader.readBool() ? reader.readDouble() : null,
      lng: reader.readBool() ? reader.readDouble() : null,
      createdAt: DateTime.fromMillisecondsSinceEpoch(reader.readInt()),
    );
  }

  @override
  void write(BinaryWriter writer, Reading obj) {
    writer.writeString(obj.machineId);
    writer.writeString(obj.machineName);
    writer.writeInt(obj.serviceIntervalH);
    writer.writeDouble(obj.hours);
    if (obj.photoPath != null) {
      writer.writeBool(true);
      writer.writeString(obj.photoPath!);
    } else {
      writer.writeBool(false);
    }
    if (obj.lat != null) {
      writer.writeBool(true);
      writer.writeDouble(obj.lat!);
    } else {
      writer.writeBool(false);
    }
    if (obj.lng != null) {
      writer.writeBool(true);
      writer.writeDouble(obj.lng!);
    } else {
      writer.writeBool(false);
    }
    writer.writeInt(obj.createdAt.millisecondsSinceEpoch);
  }
}

// =============================
// ANDROID Setup (android/app/src/main/AndroidManifest.xml)
// =============================
/*
<manifest ...>
  <uses-permission android:name="android.permission.CAMERA" />
  <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
  <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

  <application ...>
    <provider
      android:name="androidx.core.content.FileProvider"
      android:authorities="${applicationId}.fileprovider"
      android:exported="false"
      android:grantUriPermissions="true">
      <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
    </provider>
  </application>
</manifest>
*/

// android/app/src/main/res/xml/file_paths.xml
/*
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
  <external-files-path name="images" path="Pictures" />
  <cache-path name="cache" path="" />
  <files-path name="files" path="" />
</paths>
*/

// =============================
// iOS Setup (ios/Runner/Info.plist)
// =============================
/*
<key>NSCameraUsageDescription</key>
<string>Kamera wird zum Scannen von QR-Codes und Erfassen von Fotos verwendet.</string>
<key>NSLocationWhenInUseUsageDescription</key>
<string>Standort wird zur Dokumentation des Einsatzortes erfasst.</string>
*/

// =============================
// Build-Hinweise
// =============================
/*
1) Flutter installieren (3.22+), dann im Projektordner:
   flutter pub get

2) Android (APK debug):
   flutter build apk --debug
   Ausgabe: build/app/outputs/flutter-apk/app-debug.apk

3) Android (APK release, ohne Signatur für Test intern):
   flutter build apk --release

4) iOS (Simulator/Device):
   open ios/Runner.xcworkspace
   In Xcode Team/Signing setzen, dann auf Gerät bauen.

QR-Inhalt-Beispiel (so sind deine Labels gedacht):
{
  "id": "301103",
  "name": "Liebherr A900 C",
  "serviceintervall_h": 250
}
*/
