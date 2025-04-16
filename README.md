# flutter-project
// pubspec.yaml (add required dependencies)
dependencies:
  flutter:
    sdk: flutter
  firebase_core: ^2.10.0
  cloud_firestore: ^4.8.0
  flutter_bloc: ^8.1.2
  equatable: ^2.0.5

// lib/main.dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'bloc/product_bloc.dart';
import 'screens/product_list_screen.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Product Listing',
      theme: ThemeData(primarySwatch: Colors.blue),
      home: BlocProvider(
        create: (_) => ProductBloc()..add(FetchProducts()),
        child: const ProductListScreen(),
      ),
    );
  }
}

// lib/bloc/product_bloc.dart
import 'package:bloc/bloc.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:equatable/equatable.dart';

part 'product_event.dart';
part 'product_state.dart';

class ProductBloc extends Bloc<ProductEvent, ProductState> {
  final FirebaseFirestore firestore = FirebaseFirestore.instance;
  static const int limit = 10;
  DocumentSnapshot? lastDocument;
  bool isFetching = false;

  ProductBloc() : super(ProductInitial()) {
    on<FetchProducts>(_onFetchProducts);
    on<SearchProducts>(_onSearchProducts);
  }

  Future<void> _onFetchProducts(FetchProducts event, Emitter<ProductState> emit) async {
    if (isFetching) return;
    isFetching = true;
    try {
      Query query = firestore.collection('products').orderBy('createdAt', descending: true).limit(limit);

      if (lastDocument != null) {
        query = query.startAfterDocument(lastDocument!);
      }

      final snapshot = await query.get();
      if (snapshot.docs.isNotEmpty) {
        lastDocument = snapshot.docs.last;
      }

      final products = snapshot.docs.map((doc) => Product.fromFirestore(doc)).toList();
      emit(ProductLoaded(state.products + products));
    } catch (e) {
      emit(ProductError('Failed to fetch products'));
    }
    isFetching = false;
  }

  void _onSearchProducts(SearchProducts event, Emitter<ProductState> emit) async {
    try {
      final snapshot = await firestore
          .collection('products')
          .where('name', isGreaterThanOrEqualTo: event.query)
          .where('name', isLessThanOrEqualTo: '${event.query}\uf8ff')
          .get();

      final results = snapshot.docs.map((doc) => Product.fromFirestore(doc)).toList();
      emit(ProductLoaded(results));
    } catch (e) {
      emit(ProductError('Search failed'));
    }
  }
}

// lib/bloc/product_event.dart
part of 'product_bloc.dart';

abstract class ProductEvent extends Equatable {
  @override
  List<Object> get props => [];
}

class FetchProducts extends ProductEvent {}

class SearchProducts extends ProductEvent {
  final String query;

  SearchProducts(this.query);

  @override
  List<Object> get props => [query];
}

// lib/bloc/product_state.dart
part of 'product_bloc.dart';

class Product extends Equatable {
  final String id, name, imageUrl, description;
  final double price;
  final Timestamp createdAt;

  const Product({required this.id, required this.name, required this.price, required this.imageUrl, required this.description, required this.createdAt});

  factory Product.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return Product(
      id: doc.id,
      name: data['name'],
      price: data['price'].toDouble(),
      imageUrl: data['imageUrl'],
      description: data['description'],
      createdAt: data['createdAt'],
    );
  }

  @override
  List<Object?> get props => [id, name, price, imageUrl, description, createdAt];
}

abstract class ProductState extends Equatable {
  final List<Product> products;
  const ProductState(this.products);

  @override
  List<Object> get props => [products];
}

class ProductInitial extends ProductState {
  const ProductInitial() : super(const []);
}

class ProductLoaded extends ProductState {
  const ProductLoaded(List<Product> products) : super(products);
}

class ProductError extends ProductState {
  final String message;
  const ProductError(this.message) : super(const []);

  @override
  List<Object> get props => [message];
}

// lib/screens/product_list_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import '../bloc/product_bloc.dart';

class ProductListScreen extends StatefulWidget {
  const ProductListScreen({super.key});

  @override
  State<ProductListScreen> createState() => _ProductListScreenState();
}

class _ProductListScreenState extends State<ProductListScreen> {
  final ScrollController _scrollController = ScrollController();
  final TextEditingController _searchController = TextEditingController();

  @override
  void initState() {
    super.initState();
    _scrollController.addListener(_onScroll);
    _searchController.addListener(_onSearchChanged);
  }

  void _onScroll() {
    if (_scrollController.position.pixels == _scrollController.position.maxScrollExtent) {
      context.read<ProductBloc>().add(FetchProducts());
    }
  }

  void _onSearchChanged() {
    final query = _searchController.text.trim();
    if (query.isEmpty) {
      context.read<ProductBloc>().add(FetchProducts());
    } else {
      context.read<ProductBloc>().add(SearchProducts(query));
    }
  }

  @override
  void dispose() {
    _scrollController.dispose();
    _searchController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Products')),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: TextField(
              controller: _searchController,
              decoration: const InputDecoration(
                hintText: 'Search products...',
                prefixIcon: Icon(Icons.search),
              ),
            ),
          ),
          Expanded(
            child: BlocBuilder<ProductBloc, ProductState>(
              builder: (context, state) {
                if (state is ProductError) {
                  return Center(child: Text(state.message));
                } else if (state.products.isEmpty) {
                  return const Center(child: CircularProgressIndicator());
                } else {
                  return ListView.builder(
                    controller: _scrollController,
                    itemCount: state.products.length,
                    itemBuilder: (context, index) {
                      final product = state.products[index];
                      return ListTile(
                        leading: Image.network(product.imageUrl, width: 50, height: 50, fit: BoxFit.cover),
                        title: Text(product.name),
                        subtitle: Text('₹${product.price.toStringAsFixed(2)}'),
                      );
                    },
                  );
                }
              },
            ),
          )
        ],
      ),
    );
  }
}
