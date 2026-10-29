lib/screen/favorites_screen.dart
import 'package:flutter/material.dart';
import '../db/database_helper.dart';
import '../models/driver.dart';
import 'driver_details_screen.dart';

class FavoritesScreen extends StatefulWidget {
  const FavoritesScreen({super.key});
  @override
  State<FavoritesScreen> createState() => _FavoritesScreenState();
}

class _FavoritesScreenState extends State<FavoritesScreen> {
  List<Driver> _favs = [];

  @override
  void initState() {
    super.initState();
    _load();
  }

  Future<void> _load() async {
    final data = await DatabaseHelper.instance.all(onlyFav: true);
    if (mounted) setState(() => _favs = data);
  }

  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('المفضلة')),
        body: _favs.isEmpty
            ? const Center(child: Text('لا يوجد سائقون في المفضلة'))
            : ListView.builder(
                itemCount: _favs.length,
                itemBuilder: (_, i) => ListTile(
                  leading: const Icon(Icons.star, color: Colors.amber),
                  title: Text(_favs[i].fullName),
                  subtitle: Text(_favs[i].plateNumber),
                  onTap: () async {
                    await Navigator.push(
                        context,
                        MaterialPageRoute(
                            builder: (_) =>
                                DriverDetailsScreen(driver: _favs[i])));
                    _load();
                  },
                ),
              ),
      );
}

lib/screen/share_card_screen.dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:screenshot/screenshot.dart';
import 'package:share_plus/share_plus.dart';
import 'package:path_provider/path_provider.dart';
import '../models/driver.dart';

class ShareCardScreen extends StatefulWidget {
  final Driver driver;
  const ShareCardScreen({super.key, required this.driver});
  @override
  State<ShareCardScreen> createState() => _ShareCardScreenState();
}

class _ShareCardScreenState extends State<ShareCardScreen> {
  final _sc = ScreenshotController();

  Future<void> _share() async {
    final bytes = await _sc.capture(pixelRatio: 2.0);
    final dir = await getTemporaryDirectory();
    final file = File('${dir.path}/driver_${widget.driver.id}.png');
    await file.writeAsBytes(bytes);
    await Share.shareXFiles([XFile(file.path)],
        text: 'بطاقة السائق: ${widget.driver.fullName}');
  }

  @override
  Widget build(BuildContext context) {
    final d = widget.driver;
    return Scaffold(
      appBar: AppBar(title: const Text('بطاقة السائق')),
      body: Column(children: [
        Expanded(
          child: Center(
            child: Screenshot(
              controller: _sc,
              child: Container(
                width: 320,
                padding: const EdgeInsets.all(20),
                decoration: BoxDecoration(
                  gradient: const LinearGradient(
                    colors: [Color(0xFF3949AB), Color(0xFF1E88E5)],
                    begin: Alignment.topRight,
                    end: Alignment.bottomLeft,
                  ),
                  borderRadius: BorderRadius.circular(20),
                ),
                child: Column(mainAxisSize: MainAxisSize.min, children: [
                  CircleAvatar(
                      radius: 50,
                      backgroundColor: Colors.white,
                      backgroundImage: d.photoPath != null
                          ? FileImage(File(d.photoPath!))
                          : null,
                      child: d.photoPath == null
                          ? const Icon(Icons.person,
                              size: 60, color: Colors.indigo)
                          : null),
                  const SizedBox(height: 12),
                  Text(d.fullName,
                      style: const TextStyle(
                          color: Colors.white,
                          fontSize: 22,
                          fontWeight: FontWeight.bold)),
                  const SizedBox(height: 4),
                  Row(
                      mainAxisSize: MainAxisSize.min,
                      children: List.generate(
                          5,
                          (i) => Icon(
                              i < d.rating.round()
                                  ? Icons.star
                                  : Icons.star_border,
                              color: Colors.amber,
                              size: 20))),
                  const SizedBox(height: 16),
                  _cardLine(Icons.directions_car, 'اللوحة', d.plateNumber),
                  _cardLine(Icons.phone, 'الهاتف',
                      d.hidePhone ? 'الرقم مخفي' : d.phone),
                  if (d.carPhotoPath != null) ...[
                    const SizedBox(height: 12),
                    ClipRRect(
                        borderRadius: BorderRadius.circular(12),
                        child: Image.file(File(d.carPhotoPath!),
                            height: 120, fit: BoxFit.cover)),
                  ],
                ]),
              ),
            ),
          ),
        ),
        Padding(
          padding: const EdgeInsets.all(16),
          child: SizedBox(
              width: double.infinity,
              child: FilledButton.icon(
                  onPressed: _share,
                  icon: const Icon(Icons.share),
                  label: const Text('مشاركة البطاقة'))),
        ),
      ]),
    );
  }

  Widget _cardLine(IconData icon, String label, String value) => Padding(
        padding: const EdgeInsets.symmetric(vertical: 4),
        child: Row(children: [
          Icon(icon, color: Colors.white, size: 20),
          const SizedBox(width: 8),
          Text('$label: ', style: const TextStyle(color: Colors.white70)),
          Expanded(
              child: Text(value,
                  style: const TextStyle(
                      color: Colors.white, fontWeight: FontWeight.bold))),
        ]),
      );
}

lib/screen/srsh_screen.dart
import 'package:flutter/material.dart';
import '../db/database_helper.dart';
import '../models/driver.dart';
import 'driver_details_screen.dart';

class SearchScreen extends StatefulWidget {
  const SearchScreen({super.key});
  @override
  State<SearchScreen> createState() => _SearchScreenState();
}

class _SearchScreenState extends State<SearchScreen> {
  final _c = TextEditingController();
  List<Driver> _results = [];

  Future<void> _search(String q) async {
    final r = await DatabaseHelper.instance.all(query: q);
    if (mounted) setState(() => _results = r);
  }

  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('بحث')),
        body: Column(children: [
          Padding(
            padding: const EdgeInsets.all(16),
            child: TextField(
              controller: _c,
              autofocus: true,
              decoration: const InputDecoration(
                  hintText: 'الاسم، اللوحة، الهاتف...',
                  prefixIcon: Icon(Icons.search),
                  border: OutlineInputBorder()),
              onChanged: _search,
            ),
          ),
          Expanded(
              child: ListView.builder(
            itemCount: _results.length,
            itemBuilder: (_, i) {
              final d = _results[i];
              return ListTile(
                title: Text(d.fullName),
                subtitle: Text('${d.plateNumber} • ${d.phone}'),
                onTap: () => Navigator.push(
                    context,
                    MaterialPageRoute(
                        builder: (_) => DriverDetailsScreen(driver: d))),
              );
            },
          )),
        ]),
      );
}

lib/screen/driver_ditails_screen.dart
import 'dart:io';
import 'package:flutter/material.dart';
import '../models/driver.dart';
import '../db/database_helper.dart';
import 'add_edit_driver_screen.dart';
import 'share_card_screen.dart';

class DriverDetailsScreen extends StatelessWidget {
  final Driver driver;
  const DriverDetailsScreen({super.key, required this.driver});

  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: Text(driver.fullName), actions: [
          IconButton(
              icon: const Icon(Icons.share),
              onPressed: () => Navigator.push(
                  context,
                  MaterialPageRoute(
                      builder: (_) => ShareCardScreen(driver: driver)))),
          IconButton(
              icon: const Icon(Icons.edit),
              onPressed: () async {
                await Navigator.push(
                    context,
                    MaterialPageRoute(
                        builder: (_) =>
                            AddEditDriverScreen(driver: driver)));
                if (context.mounted) Navigator.pop(context);
              }),
          IconButton(
              icon: const Icon(Icons.delete, color: Colors.red),
              onPressed: () async {
                final ok = await showDialog<bool>(
                    context: context,
                    builder: (_) => AlertDialog(
                          title: const Text('حذف؟'),
                          content: const Text('هل تريد حذف هذا السائق؟'),
                          actions: [
                            TextButton(
                                onPressed: () =>
                                    Navigator.pop(context, false),
                                child: const Text('إلغاء')),
                            TextButton(
                                onPressed: () =>
                                    Navigator.pop(context, true),
                                child: const Text('حذف')),
                          ],
                        ));
                if (ok == true) {
                  await DatabaseHelper.instance.delete(driver.id!);
                  if (context.mounted) Navigator.pop(context);
                }
              }),
        ]),
        body: ListView(padding: const EdgeInsets.all(16), children: [
          Center(
              child: CircleAvatar(
                  radius: 60,
                  backgroundImage: driver.photoPath != null
                      ? FileImage(File(driver.photoPath!))
                      : null,
                  child: driver.photoPath == null
                      ? const Icon(Icons.person, size: 60)
                      : null)),
          const SizedBox(height: 16),
          Center(
              child: Text(driver.fullName,
                  style: const TextStyle(
                      fontSize: 24, fontWeight: FontWeight.bold))),
          const SizedBox(height: 8),
          Center(
              child: Row(mainAxisSize: MainAxisSize.min, children: [
            ...List.generate(
                5,
                (i) => Icon(
                    i < driver.rating.round()
                        ? Icons.star
                        : Icons.star_border,
                    color: Colors.amber))
          ])),
          const Divider(height: 32),
          _row(Icons.directions_car, 'رقم اللوحة', driver.plateNumber),
          _row(Icons.phone, 'الهاتف',
              driver.hidePhone ? '•••• ••••' : driver.phone),
          if (driver.notes != null && driver.notes!.isNotEmpty)
            _row(Icons.notes, 'ملاحظات', driver.notes!),
          if (driver.carPhotoPath != null) ...[
            const SizedBox(height: 16),
            const Text('صورة السيارة',
                style: TextStyle(fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            ClipRRect(
                borderRadius: BorderRadius.circular(12),
                child: Image.file(File(driver.carPhotoPath!),
                    height: 200, fit: BoxFit.cover)),
          ],
          const SizedBox(height: 24),
          FilledButton.icon(
            onPressed: () => Navigator.push(
                context,
                MaterialPageRoute(
                    builder: (_) => ShareCardScreen(driver: driver))),
            icon: const Icon(Icons.share),
            label: const Text('مشاركة بطاقة السائق'),
          ),
        ]),
      );

  Widget _row(IconData i, String label, String value) => ListTile(
        leading: Icon(i, color: Colors.indigo),
        title: Text(label,
            style: const TextStyle(fontSize: 12, color: Colors.grey)),
        subtitle: Text(value, style: const TextStyle(fontSize: 16)),
      );
}

lib/screen/add_edit_driver_screen.art
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:image_picker/image_picker.dart';
import '../db/database_helper.dart';
import '../models/driver.dart';

class AddEditDriverScreen extends StatefulWidget {
  final Driver? driver;
  const AddEditDriverScreen({super.key, this.driver});
  @override
  State<AddEditDriverScreen> createState() => _AddEditDriverScreenState();
}

class _AddEditDriverScreenState extends State<AddEditDriverScreen> {
  final _formKey = GlobalKey<FormState>();
  late TextEditingController _fn, _ln, _plate, _phone, _notes;
  String? _photo, _carPhoto;
  bool _hidePhone = false;
  double _rating = 0;
  bool _isFav = false;
  final _picker = ImagePicker();

  @override
  void initState() {
    super.initState();
    final d = widget.driver;
    _fn = TextEditingController(text: d?.firstName ?? '');
    _ln = TextEditingController(text: d?.lastName ?? '');
    _plate = TextEditingController(text: d?.plateNumber ?? '');
    _phone = TextEditingController(text: d?.phone ?? '');
    _notes = TextEditingController(text: d?.notes ?? '');
    _photo = d?.photoPath;
    _carPhoto = d?.carPhotoPath;
    _hidePhone = d?.hidePhone ?? false;
    _rating = d?.rating ?? 0;
    _isFav = d?.isFavorite ?? false;
  }

  Future<void> _pick(bool isDriver) async {
    final x = await _picker.pickImage(
        source: ImageSource.gallery, imageQuality: 75);
    if (x != null) {
      setState(() => isDriver ? _photo = x.path : _carPhoto = x.path);
    }
  }

  Future<void> _save() async {
    if (!_formKey.currentState!.validate()) return;
    final d = Driver(
      id: widget.driver?.id,
      firstName: _fn.text.trim(),
      lastName: _ln.text.trim(),
      plateNumber: _plate.text.trim(),
      phone: _phone.text.trim(),
      notes: _notes.text.trim(),
      photoPath: _photo,
      carPhotoPath: _carPhoto,
      hidePhone: _hidePhone,
      rating: _rating,
      isFavorite: _isFav,
    );
    if (d.id == null) {
      await DatabaseHelper.instance.insert(d);
    } else {
      await DatabaseHelper.instance.update(d);
    }
    if (mounted) Navigator.pop(context);
  }

  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(
            title: Text(
                widget.driver == null ? 'إضافة سائق' : 'تعديل سائق')),
        body: Form(
          key: _formKey,
          child: ListView(padding: const EdgeInsets.all(16), children: [
            Row(children: [
              _photoTile('صورة السائق', _photo, () => _pick(true)),
              const SizedBox(width: 16),
              _photoTile('صورة السيارة', _carPhoto, () => _pick(false)),
            ]),
            const SizedBox(height: 16),
            TextFormField(
                controller: _fn,
                decoration: const InputDecoration(
                    labelText: 'الاسم', border: OutlineInputBorder()),
                validator: (v) => v!.isEmpty ? 'مطلوب' : null),
            const SizedBox(height: 12),
            TextFormField(
                controller: _ln,
                decoration: const InputDecoration(
                    labelText: 'الشهرة', border: OutlineInputBorder())),
            const SizedBox(height: 12),
            TextFormField(
                controller: _plate,
                decoration: const InputDecoration(
                    labelText: 'رقم اللوحة', border: OutlineInputBorder()),
                validator: (v) => v!.isEmpty ? 'مطلوب' : null),
            const SizedBox(height: 12),
            TextFormField(
                controller: _phone,
                keyboardType: TextInputType.phone,
                decoration: const InputDecoration(
                    labelText: 'رقم الهاتف', border: OutlineInputBorder()),
                validator: (v) => v!.isEmpty ? 'مطلوب' : null),
            SwitchListTile(
                title: const Text('إخفاء الرقم عند المشاركة'),
                value: _hidePhone,
                onChanged: (v) => setState(() => _hidePhone = v)),
            const SizedBox(height: 8),
            const Text('تقييم السائق'),
            Slider(
                value: _rating,
                max: 5,
                divisions: 10,
                label: _rating.toStringAsFixed(1),
                onChanged: (v) => setState(() => _rating = v)),
            SwitchListTile(
                title: const Text('إضافة للمفضلة'),
                value: _isFav,
                onChanged: (v) => setState(() => _isFav = v)),
            TextFormField(
                controller: _notes,
                maxLines: 3,
                decoration: const InputDecoration(
                    labelText: 'ملاحظات', border: OutlineInputBorder())),
            const SizedBox(height: 20),
            FilledButton.icon(
                onPressed: _save,
                icon: const Icon(Icons.save),
                label: const Text('حفظ')),
          ]),
        ),
      );

  Widget _photoTile(String label, String? path, VoidCallback onTap) =>
      Expanded(
        child: InkWell(
          onTap: onTap,
          child: Container(
            height: 120,
            decoration: BoxDecoration(
                border: Border.all(), borderRadius: BorderRadius.circular(12)),
            child: path == null
                ? Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                    const Icon(Icons.camera_alt, size: 40),
                    Text(label)
                  ])
                : ClipRRect(
                    borderRadius: BorderRadius.circular(12),
                    child: Image.file(File(path),
                        fit: BoxFit.cover, width: double.infinity)),
          ),
        ),
      );
}

lib/screens/drivers_list_screens.dart

import 'dart:io';
import 'package:flutter/material.dart';
import '../db/database_helper.dart';
import '../models/driver.dart';
import 'add_edit_driver_screen.dart';
import 'driver_details_screen.dart';

class DriversListScreen extends StatefulWidget {
  const DriversListScreen({super.key});
  @override
  State<DriversListScreen> createState() => _DriversListScreenState();
}

class _DriversListScreenState extends State<DriversListScreen> {
  List<Driver> _drivers = [];

  @override
  void initState() {
    super.initState();
    _load();
  }

  Future<void> _load() async {
    final data = await DatabaseHelper.instance.all();
    if (mounted) setState(() => _drivers = data);
  }

  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('السائقون')),
        body: _drivers.isEmpty
            ? const Center(child: Text('لا يوجد سائقون بعد'))
            : ListView.builder(
                itemCount: _drivers.length,
                itemBuilder: (_, i) {
                  final d = _drivers[i];
                  return ListTile(
                    leading: CircleAvatar(
                      backgroundImage: d.photoPath != null
                          ? FileImage(File(d.photoPath!))
                          : null,
                      child: d.photoPath == null
                          ? Text(d.firstName.isNotEmpty ? d.firstName[0] : '?')
                          : null,
                    ),
                    title: Text(d.fullName),
                    subtitle: Text(
                        '${d.plateNumber} • ⭐ ${d.rating.toStringAsFixed(1)}'),
                    trailing: d.isFavorite
                        ? const Icon(Icons.star, color: Colors.amber)
                        : null,
                    onTap: () async {
                      await Navigator.push(
                          context,
                          MaterialPageRoute(
                              builder: (_) => DriverDetailsScreen(driver: d)));
                      _load();
                    },
                  );
                },
              ),
        floatingActionButton: FloatingActionButton.extended(
          onPressed: () async {
            await Navigator.push(context,
                MaterialPageRoute(builder: (_) => const AddEditDriverScreen()));
            _load();
          },
          icon: const Icon(Icons.add),
          label: const Text('إضافة سائق'),
        ),
      );
}
Lib/screens/homes_screen.dart
import 'package:flutter/material.dart';
import 'drivers_list_screen.dart';
import 'favorites_screen.dart';
import 'search_screen.dart';
import 'settings_screen.dart';

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final tiles = [
      _Tile(Icons.people, 'كل السائقين', () => const DriversListScreen()),
      _Tile(Icons.star, 'المفضلة', () => const FavoritesScreen()),
      _Tile(Icons.search, 'بحث', () => const SearchScreen()),
      _Tile(Icons.settings, 'الإعدادات', () => const SettingsScreen()),
    ];
    return Scaffold(
      appBar: AppBar(title: const Text('الرئيسية'), centerTitle: true),
      body: GridView.count(
        crossAxisCount: 2,
        padding: const EdgeInsets.all(16),
        mainAxisSpacing: 16,
        crossAxisSpacing: 16,
        children: tiles
            .map((t) => Card(
                  child: InkWell(
                    onTap: t.onTap,
                    borderRadius: BorderRadius.circular(12),
                    child: Column(
                        mainAxisAlignment: MainAxisAlignment.center,
                        children: [
                          Icon(t.icon,
                              size: 56,
                              color: Theme.of(context).colorScheme.primary),
                          const SizedBox(height: 12),
                          Text(t.label, style: const TextStyle(fontSize: 18)),
                        ]),
                  ),
                ))
            .toList(),
      ),
    );
  }
}

class _Tile {
  final IconData icon;
  final String label;
  final VoidCallback onTap;
  _Tile(this.icon, this.label, this.onTap);
}

Lib/screens/splash/screens.dart
import 'package:flutter/material.dart';
import 'home_screen.dart';

class SplashScreen extends StatefulWidget {
  const SplashScreen({super.key});
  @override
  State<SplashScreen> createState() => _SplashScreenState();
}

class _SplashScreenState extends State<SplashScreen> {
  @override
  void initState() {
    super.initState();
    Future.delayed(const Duration(seconds: 2), () {
      if (mounted) {
        Navigator.pushReplacement(
            context, MaterialPageRoute(builder: (_) => const HomeScreen()));
      }
    });
  }

  @override
  Widget build(BuildContext context) => Scaffold(
        body: Center(
          child: Column(mainAxisSize: MainAxisSize.min, children: [
            Icon(Icons.local_taxi,
                size: 100, color: Theme.of(context).colorScheme.primary),
            const SizedBox(height: 16),
            const Text('دليل السائقين',
                style: TextStyle(fontSize: 28, fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            const CircularProgressIndicator(),
          ]),
        ),
      );
}
Lib/db/database/helper.dart
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';
import '../models/driver.dart';

class DatabaseHelper {
  static final DatabaseHelper instance = DatabaseHelper._();
  DatabaseHelper._();
  static Database? _db;

  Future<Database> get db async {
    _db ??= await _init();
    return _db!;
  }

  Future<Database> _init() async {
    final path = join(await getDatabasesPath(), 'drivers.db');
    return openDatabase(path, version: 1, onCreate: (db, v) async {
      await db.execute('''
        CREATE TABLE drivers(
          id INTEGER PRIMARY KEY AUTOINCREMENT,
          firstName TEXT, lastName TEXT,
          photoPath TEXT, carPhotoPath TEXT,
          plateNumber TEXT, phone TEXT,
          hidePhone INTEGER, rating REAL,
          isFavorite INTEGER, notes TEXT, createdAt TEXT
        )
      ''');
    });
  }

  Future<int> insert(Driver d) async =>
      (await db).insert('drivers', d.toMap());

  Future<int> update(Driver d) async =>
      (await db).update('drivers', d.toMap(), where: 'id=?', whereArgs: [d.id]);

  Future<int> delete(int id) async =>
      (await db).delete('drivers', where: 'id=?', whereArgs: [id]);

  Future<List<Driver>> all({String? query, bool onlyFav = false}) async {
    String? where;
    List<dynamic> args = [];
    if (onlyFav) where = 'isFavorite=1';
    if (query != null && query.isNotEmpty) {
      final q = '%$query%';
      final cond =
          '(firstName LIKE ? OR lastName LIKE ? OR plateNumber LIKE ? OR phone LIKE ?)';
      where = where == null ? cond : '$where AND $cond';
      args = [q, q, q, q];
    }
    final rows = await (await db).query('drivers',
        where: where, whereArgs: args, orderBy: 'firstName ASC');
    return rows.map(Driver.fromMap).toList();
  }
}
Lib/models/driver.dart
class Driver {
  int? id;
  String firstName;
  String lastName;
  String? photoPath;
  String? carPhotoPath;
  String plateNumber;
  String phone;
  bool hidePhone;
  double rating;
  bool isFavorite;
  String? notes;
  DateTime createdAt;

  Driver({
    this.id,
    required this.firstName,
    required this.lastName,
    this.photoPath,
    this.carPhotoPath,
    required this.plateNumber,
    required this.phone,
    this.hidePhone = false,
    this.rating = 0,
    this.isFavorite = false,
    this.notes,
    DateTime? createdAt,
  }) : createdAt = createdAt ?? DateTime.now();

  String get fullName => '$firstName $lastName';

  Map<String, dynamic> toMap() => {
        'id': id,
        'firstName': firstName,
        'lastName': lastName,
        'photoPath': photoPath,
        'carPhotoPath': carPhotoPath,
        'plateNumber': plateNumber,
        'phone': phone,
        'hidePhone': hidePhone ? 1 : 0,
        'rating': rating,
        'isFavorite': isFavorite ? 1 : 0,
        'notes': notes,
        'createdAt': createdAt.toIso8601String(),
      };

  factory Driver.fromMap(Map<String, dynamic> m) => Driver(
        id: m['id'],
        firstName: m['firstName'],
        lastName: m['lastName'],
        photoPath: m['photoPath'],
        carPhotoPath: m['carPhotoPath'],
        plateNumber: m['plateNumber'],
        phone: m['phone'],
        hidePhone: m['hidePhone'] == 1,
        rating: (m['rating'] as num).toDouble(),
        isFavorite: m['isFavorite'] == 1,
        notes: m['notes'],
        createdAt: DateTime.parse(m['createdAt']),
      );
}

<!-- Banner Image -->

<p align="center">
  <a href="https://expo.dev/">
    <img alt="Expo logo" height="128" src="./.github/resources/banner.png">
    <h1 align="center">Expo</h1>
  </a>
</p>

<p align="center">
   <a aria-label="SDK version" href="https://www.npmjs.com/package/expo" target="_blank">
    <img alt="Expo SDK version" src="https://img.shields.io/npm/v/expo.svg?style=flat-square&label=SDK&labelColor=000000&color=4630EB" />
  </a>
  <a aria-label="Chat or ask a question" href="https://chat.expo.dev" target="_blank">
    <img alt="Chat or ask a question" src="https://img.shields.io/discord/695411232856997968.svg?style=flat-square&labelColor=000000&color=4630EB&logo=discord&logoColor=FFFFFF&label=Chat%20with%20us" />
  </a>
  <a aria-label="Expo is free to use" href="https://github.com/expo/expo/blob/main/LICENSE" target="_blank">
    <img alt="License: MIT" src="https://img.shields.io/badge/License-MIT-success.svg?style=flat-square&color=33CC12" target="_blank" />
  </a>
  <a aria-label="expo downloads" href="http://www.npmtrends.com/expo" target="_blank">
    <img alt="Downloads" src="https://img.shields.io/npm/dm/expo.svg?style=flat-square&labelColor=gray&color=33CC12&label=Downloads" />
  </a>
</p>

<p align="center">
  <a aria-label="try expo with snack" href="https://snack.expo.dev"><b>Try Expo in the Browser</b></a>
&ensp;•&ensp;
  <a aria-label="expo documentation" href="https://docs.expo.dev">Read the Documentation</a>
&ensp;•&ensp;
  <a aria-label="expo documentation" href="https://expo.dev/blog">Learn more on our blog</a>
&ensp;•&ensp;
  <a aria-label="expo documentation" href="https://expo.canny.io/feature-requests">Request a feature</a>
</p>

<h6 align="center">Follow us on</h6>
<p align="center">
  <a aria-label="Follow @expo on X" href="https://x.com/intent/follow?screen_name=expo" target="_blank">
    <img alt="Expo on X" src="https://img.shields.io/badge/X-000000?style=for-the-badge&logo=x&logoColor=white" target="_blank" />
  </a>&nbsp;
  <a aria-label="Follow @expo on GitHub" href="https://github.com/expo" target="_blank">
    <img alt="Expo on GitHub" src="https://img.shields.io/badge/GitHub-222222?style=for-the-badge&logo=github&logoColor=white" target="_blank" />
  </a>&nbsp;
  <a aria-label="Follow @expo on Reddit" href="https://www.reddit.com/r/expo/" target="_blank">
    <img alt="Expo on Reddit" src="https://img.shields.io/badge/Reddit-FF4500?style=for-the-badge&logo=reddit&logoColor=white" target="_blank" />
  </a>&nbsp;
  <a aria-label="Follow @expo on Bluesky" href="https://bsky.app/profile/expo.dev" target="_blank">
    <img alt="Expo on Bluesky" src="https://img.shields.io/badge/Bluesky-1DA1F2?style=for-the-badge&logo=bluesky&logoColor=white" target="_blank" />
  </a>&nbsp;
  <a aria-label="Follow @expo on LinkedIn" href="https://www.linkedin.com/company/expo-dev" target="_blank">
    <img alt="Expo on LinkedIn" src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white" target="_blank" />
  </a>
</p>

## Introduction

Expo is an open-source platform for making universal native apps that run on Android, iOS, and the web. It includes a universal runtime and libraries that let you build native apps by writing React and JavaScript.

This repository includes the Expo SDK, Modules API, Go app, CLI, Router, documentation, and various other supporting tools. [Expo Application Services (EAS)](https://expo.dev/eas) is a platform of hosted services that are deeply integrated with Expo open source tools. EAS helps you build, ship, and iterate on your app as an individual or a team.

Read the [Expo Community Guidelines](https://expo.dev/guidelines) before interacting in the repository. Thank you for helping keep the Expo community open and welcoming!

## Table of contents

- [📚 Documentation](#-documentation)
- [🗺 Project Layout](#-project-layout)
- [🏅 Badges](#-badges)
- [👏 Contributing](#-contributing)
- [❓ FAQ](#-faq)
- [💙 The Team](#-the-team)
- [License](#license)

## 📚 Documentation

<p>Learn about building and deploying universal apps <a aria-label="expo documentation" href="https://docs.expo.dev">in our official docs!</a></p>

- [Getting Started](https://docs.expo.dev/)
- [API Reference](https://docs.expo.dev/versions/latest/)
- [Using Custom Native Modules](https://docs.expo.dev/workflow/customizing/)

## 🗺 Project Layout

- [`packages`](/packages) All the source code for Expo modules, if you want to edit a library or just see how it works this is where you'll find it.
- [`apps`](/apps) This is where you can find Expo projects which are linked to the development modules. You'll do most of your testing in here.
- [`apps/expo-go`](/apps/expo-go) This is where you can find the source code for Expo Go.
- [`apps/expo-go/ios/Exponent.xcworkspace`](/apps/expo-go/ios) is the Xcode workspace. When developing iOS, always open this instead of `Exponent.xcodeproj` because the workspace also loads the CocoaPods dependencies.
- [`docs`](/docs) The source code for **https://docs.expo.dev**
- [`templates`](/templates) The template projects you get when you run `npx create-expo-app`
- [`react-native-lab`](/react-native-lab) This is our fork of `react-native` used to build Expo Go.
- [`guides`](/guides) In-depth tutorials for advanced topics like contributing to the client.
- [`tools`](/tools) contain build and configuration tools.
- [`template-files`](/template-files) contains templates for files that require private keys. They are populated using the keys in `template-files/keys.json`.
- [`template-files/ios/dependencies.json`](/template-files/ios/dependencies.json) specifies the CocoaPods dependencies of the app.

## 🏅 Badges

Let everyone know your app can be run instantly in the _Expo Go_ app!
<br/>

[![runs with Expo Go](https://img.shields.io/badge/Runs%20with%20Expo%20Go-000.svg?style=flat-square&logo=EXPO&labelColor=f3f3f3&logoColor=000)](https://expo.dev/client)

[![runs with Expo Go](https://img.shields.io/badge/Runs%20with%20Expo%20Go-4630EB.svg?style=flat-square&logo=EXPO&labelColor=f3f3f3&logoColor=000)](https://expo.dev/client)

```md
[![runs with Expo Go](https://img.shields.io/badge/Runs%20with%20Expo%20Go-000.svg?style=flat-square&logo=EXPO&labelColor=f3f3f3&logoColor=000)](https://expo.dev/client)

[![runs with Expo Go](https://img.shields.io/badge/Runs%20with%20Expo%20Go-4630EB.svg?style=flat-square&logo=EXPO&labelColor=f3f3f3&logoColor=000)](https://expo.dev/client)
```

## 👏 Contributing

If you like Expo and want to help make it better then check out our [contributing guide](/CONTRIBUTING.md)! Check out the [CLI package](https://github.com/expo/expo/tree/main/packages/%40expo/cli) to work on the Expo CLI.

## ❓ FAQ

If you have questions about Expo and want answers, then check out our [Frequently Asked Questions](https://docs.expo.dev/faq/)!

If you still have questions you can ask them on our [Discord and Forums](https://chat.expo.dev) or X [@expo](https://x.com/expo).

## 💙 The Team

Curious about who makes Expo? Here are our [team members](https://expo.dev/about)!

## License

The Expo source code is made available under the [MIT license](LICENSE). Some of the dependencies are licensed differently, with the BSD license, for example.

<img alt="Star the Expo repo on GitHub to support the project" src="https://user-images.githubusercontent.com/9664363/185428788-d762fd5d-97b3-4f59-8db7-f72405be9677.gif" width="50%">
