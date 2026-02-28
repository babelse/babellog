// ---------------------------------------------------------------------------
// PAGE DE CHAT (BabelLOG AI - Sécurisée avec Auth0 & Nom dynamique)
// ---------------------------------------------------------------------------

class ChatMatierePage extends StatefulWidget {
  final String nomMatiere;

  const ChatMatierePage({super.key, required this.nomMatiere});

  @override
  State<ChatMatierePage> createState() => _ChatMatierePageState();
}

class _ChatMatierePageState extends State<ChatMatierePage> {
  final TextEditingController _messageController = TextEditingController();
  final ScrollController _scrollController = ScrollController();
  final ImagePicker _picker = ImagePicker();
  
  File? _selectedImage;
  final String groqApiKey = "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"; 
  final List<Map<String, dynamic>> _messages = [];
  bool _isLoading = false;

  // --- VARIABLES D'AUTHENTIFICATION ---
  late Auth0 auth0;
  bool _isAuthChecking = true; // Pour afficher un chargement au début
  bool _isAuthenticated = false; // Savoir s'il est connecté
  String _prenomUtilisateur = ""; // Pour stocker son vrai nom

  @override
  void initState() {
    super.initState();
    auth0 = Auth0('babellog.us.auth0.com', 'lCnbOzzHBOaAulOIh5jMlz1LlQZXH52x');
    _verifierConnexion(); // 🟢 On vérifie qui est là avant de tout afficher
  }

  // --- VÉRIFIER L'IDENTITÉ DE L'UTILISATEUR ---
  Future<void> _verifierConnexion() async {
    try {
      if (await auth0.credentialsManager.hasValidCredentials()) {
        final credentials = await auth0.credentialsManager.credentials();
        
        // On récupère le nom ou pseudo renvoyé par Google/Apple
        String rawName = credentials.user.name ?? credentials.user.nickname ?? "Étudiant";
        
        // Si c'est un email (ex: ahiman@gmail.com), on coupe pour ne garder que "ahiman"
        if (rawName.contains('@')) {
          rawName = rawName.split('@')[0];
        }
        
        // On met la première lettre en majuscule pour faire propre
        if (rawName.isNotEmpty) {
          rawName = rawName[0].toUpperCase() + rawName.substring(1).toLowerCase();
        }

        setState(() {
          _prenomUtilisateur = rawName;
          _isAuthenticated = true;
          _isAuthChecking = false;
        });
        
        // On charge l'historique uniquement s'il est autorisé à être ici
        _loadHistory(); 
      } else {
        setState(() {
          _isAuthenticated = false;
          _isAuthChecking = false;
        });
      }
    } catch (e) {
      setState(() {
        _isAuthenticated = false;
        _isAuthChecking = false;
      });
    }
  }

  Future<void> _loadHistory() async {
    final prefs = await SharedPreferences.getInstance();
    final String? rawJson = prefs.getString('chat_${widget.nomMatiere}');
    if (rawJson != null) {
      List<dynamic> decoded = jsonDecode(rawJson);
      setState(() {
        _messages.clear();
        for (var m in decoded) {
          _messages.add({
            "role": m["role"],
            "text": m["text"],
            "image": m["imagePath"] != null ? File(m["imagePath"]) : null,
            "timestamp": m["timestamp"],
          });
        }
      });
      WidgetsBinding.instance.addPostFrameCallback((_) => _scrollToBottom());
    }
  }

  Future<void> _saveHistory() async {
    final prefs = await SharedPreferences.getInstance();
    List<Map<String, dynamic>> toSave = _messages.map((m) {
      return {
        "role": m["role"],
        "text": m["text"],
        "imagePath": m["image"] != null ? (m["image"] as File).path : null,
        "timestamp": m["timestamp"] ?? DateTime.now().toIso8601String(),
      };
    }).toList();
    await prefs.setString('chat_${widget.nomMatiere}', jsonEncode(toSave));
  }

  Future<void> _pickImage() async {
    final XFile? image = await _picker.pickImage(source: ImageSource.gallery, imageQuality: 70);
    if (image != null) {
      setState(() {
        _selectedImage = File(image.path);
      });
    }
  }

  Future<void> _sendMessage() async {
    final text = _messageController.text.trim();
    if (text.isEmpty && _selectedImage == null) return;

    final File? imageToSend = _selectedImage;

    setState(() {
      _messages.add({
        "role": "user", 
        "text": text,
        "image": imageToSend,
        "timestamp": DateTime.now().toIso8601String()
      });
      _isLoading = true;
      _selectedImage = null; 
    });
    
    _saveHistory(); 
    _messageController.clear();
    _scrollToBottom();

    try {
      // On informe aussi l'IA du nom de l'utilisateur pour qu'elle puisse être plus personnelle !
      final String systemPrompt = """Tu es un professeur expert en ${widget.nomMatiere}. Tu discutes avec un étudiant nommé $_prenomUtilisateur.
RÈGLES DE FORMATAGE OBLIGATOIRES :
1) STRUCTURE GÉNÉRALE : Toujours organiser la réponse avec des titres clairs (# Titre principal, ## Sous-section, ### Sous-niveau si nécessaire). Aérer le texte.
2) DÉFINITIONS : Si le sujet implique une définition, créer une section : '## 📘 Définition'. La définition doit être claire, précise et formelle.
3) EXPLICATIONS : Utiliser des listes à puces, des listes numérotées, des séparateurs visuels (---), et des étapes claires si processus.
4) TABLEAUX : Si comparaison ou données structurées → utiliser tableau ASCII/Markdown.
5) FORMULES MATHÉMATIQUES : Toujours écrire les formules en LaTeX (Inline : \$ ... \$, Bloc : \$\$ ... \$\$).
6) PROCESSUS / MÉTHODES : Utiliser une structuration par étapes numérotées (1. Étape 1, 2. Étape 2...).
7) SI LA QUESTION EST SIMPLE : Réponse courte mais structurée avec au minimum un mini titre et une explication organisée.
8) SI LA QUESTION EST TECHNIQUE : Ajouter définitions, modélisation si pertinente, exemple concret, conclusion synthétique.
9) STYLE : Ton clair, académique, précis. Pas de texte vague. Pas de blocs massifs non structurés. Pas d’emoji sauf si explicitement demandé.
10) OBJECTIF : Chaque réponse doit ressembler à un mini-cours bien organisé.""";

      List<Map<String, dynamic>> apiMessages = [
        {"role": "system", "content": systemPrompt}
      ];

      for (var msg in _messages) {
        if (msg['role'] == 'user') {
          if (msg['image'] != null) {
            final bytes = await (msg['image'] as File).readAsBytes();
            final base64Image = base64Encode(bytes);
            
            apiMessages.add({
              "role": "user",
              "content": [
                {"type": "text", "text": msg['text'].toString().isEmpty ? "Peux-tu m'expliquer et résoudre ce qu'il y a sur cette image ?" : msg['text']},
                {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,$base64Image"}}
              ]
            });
          } else {
            apiMessages.add({"role": "user", "content": msg['text']});
          }
        } else {
          apiMessages.add({"role": "assistant", "content": msg['text']});
        }
      }

      bool hasImageInHistory = _messages.any((msg) => msg['image'] != null);
      String groqModel = hasImageInHistory 
          ? 'meta-llama/llama-4-scout-17b-16e-instruct' 
          : 'llama-3.1-8b-instant';

      final response = await http.post(
        Uri.parse('https://api.groq.com/openai/v1/chat/completions'),
        headers: {
          'Content-Type': 'application/json',
          'Authorization': 'Bearer $groqApiKey',
        },
        body: jsonEncode({
          "model": groqModel,
          "messages": apiMessages,
          "temperature": 0.5,
        }),
      );

      if (response.statusCode == 200) {
        final data = jsonDecode(utf8.decode(response.bodyBytes));
        final aiResponse = data['choices'][0]['message']['content'];
        
        setState(() {
          _messages.add({
            "role": "assistant", 
            "text": aiResponse, 
            "image": null,
            "timestamp": DateTime.now().toIso8601String() 
          });
        });
        
        _saveHistory(); 

      } else {
        setState(() {
          _messages.add({"role": "assistant", "text": "Erreur API (${response.statusCode}) : ${response.body}", "image": null});
        });
      }
    } catch (e) {
      setState(() {
        _messages.add({"role": "assistant", "text": "Impossible de se connecter au serveur IA : $e", "image": null});
      });
    } finally {
      setState(() {
        _isLoading = false;
      });
      _scrollToBottom();
    }
  }

  void _scrollToBottom() {
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (_scrollController.hasClients) {
        _scrollController.animateTo(
          _scrollController.position.maxScrollExtent,
          duration: const Duration(milliseconds: 300),
          curve: Curves.easeOut,
        );
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    // 🟢 1. SI ON VÉRIFIE L'IDENTITÉ, ON AFFICHE UN CHARGEMENT
    if (_isAuthChecking) {
      return const Scaffold(
        backgroundColor: Colors.white,
        body: Center(child: CircularProgressIndicator(color: Colors.red)),
      );
    }

    // 🟢 2. SI L'UTILISATEUR N'EST PAS CONNECTÉ, ON BLOQUE L'ACCÈS
    if (!_isAuthenticated) {
      return Scaffold(
        backgroundColor: const Color(0xFFF8F9FA),
        body: Center(
          child: Padding(
            padding: const EdgeInsets.all(30.0),
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                const Icon(Icons.lock_outline, size: 80, color: Colors.grey),
                const SizedBox(height: 20),
                const Text(
                  "Accès Refusé",
                  style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold, color: Colors.black87),
                ),
                const SizedBox(height: 15),
                const Text(
                  "Vous devez vous connecter à votre compte pour accéder à l'Intelligence Artificielle BabelLOG.",
                  textAlign: TextAlign.center,
                  style: TextStyle(color: Colors.black54, fontSize: 16, height: 1.4),
                ),
                const SizedBox(height: 40),
                SizedBox(
                  width: double.infinity,
                  height: 55,
                  child: ElevatedButton(
                    onPressed: () => Navigator.pop(context),
                    style: ElevatedButton.styleFrom(
                      backgroundColor: Colors.red.shade500,
                      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(15)),
                      elevation: 5,
                      shadowColor: Colors.red.withValues(alpha: 0.4),
                    ),
                    child: const Text("Retour à l'accueil", style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold, color: Colors.white)),
                  ),
                ),
              ],
            ),
          ),
        ),
      );
    }

    // 🟢 3. S'IL EST CONNECTÉ, ON AFFICHE LE CHAT NORMAL
    return Scaffold(
      body: Container(
        decoration: const BoxDecoration(
          gradient: LinearGradient(
            begin: Alignment.topCenter,
            end: Alignment.bottomCenter,
            colors: [Colors.white, Color(0xFFFDEBEC)],
          ),
        ),
        child: SafeArea(
          child: Column(
            children: [
              // --- HEADER ---
              Padding(
                padding: const EdgeInsets.symmetric(horizontal: 10.0, vertical: 10.0),
                child: Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    IconButton(
                      icon: const Icon(Icons.arrow_back_ios, color: Colors.black),
                      onPressed: () => Navigator.pop(context),
                    ),
                    Column(
                      children: [
                        Row(
                          children: [
                            const Text("BabelLOG", style: TextStyle(fontWeight: FontWeight.w900, fontSize: 18, color: Colors.black)),
                            const SizedBox(width: 5),
                            Icon(Icons.info_outline, size: 18, color: Colors.grey.shade800),
                          ],
                        ),
                        const Text("Beta 1", style: TextStyle(fontSize: 11, color: Colors.grey, fontWeight: FontWeight.bold)),
                      ],
                    ),
                    TextButton(
                      onPressed: () => Navigator.pop(context),
                      child: const Text("Quitter", style: TextStyle(color: Colors.red, fontWeight: FontWeight.bold, fontSize: 16)),
                    ),
                  ],
                ),
              ),

              // --- CORPS CENTRAL ---
              Expanded(
                child: _messages.isEmpty ? _buildWelcomeScreen() : _buildChatList(),
              ),

              // --- APERCU DE L'IMAGE SELECTIONNEE (AVANT ENVOI) ---
              if (_selectedImage != null)
                Container(
                  margin: const EdgeInsets.symmetric(horizontal: 20, vertical: 5),
                  alignment: Alignment.centerLeft,
                  child: Stack(
                    clipBehavior: Clip.none,
                    children: [
                      ClipRRect(
                        borderRadius: BorderRadius.circular(15),
                        child: Image.file(_selectedImage!, width: 80, height: 80, fit: BoxFit.cover),
                      ),
                      Positioned(
                        top: -10,
                        right: -10,
                        child: GestureDetector(
                          onTap: () => setState(() => _selectedImage = null),
                          child: Container(
                            padding: const EdgeInsets.all(4),
                            decoration: const BoxDecoration(color: Colors.red, shape: BoxShape.circle),
                            child: const Icon(Icons.close, color: Colors.white, size: 14),
                          ),
                        ),
                      ),
                    ],
                  ),
                ),

              // --- ZONE DE SAISIE ---
              Padding(
                padding: const EdgeInsets.all(20.0),
                child: Column(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    if (_messages.isEmpty)
                      RichText(
                        text: const TextSpan(
                          text: "En parlant ici, tu acceptes nos ",
                          style: TextStyle(color: Colors.grey, fontSize: 12),
                          children: [
                            TextSpan(text: "conditions", style: TextStyle(color: Colors.red, fontWeight: FontWeight.bold)),
                          ],
                        ),
                      ),
                    if (_messages.isEmpty) const SizedBox(height: 15),
                    
                    Container(
                      decoration: BoxDecoration(
                        color: Colors.white,
                        borderRadius: BorderRadius.circular(30),
                        boxShadow: [
                          BoxShadow(color: Colors.red.withValues(alpha: 0.5), blurRadius: 15, offset: const Offset(0, 5))
                        ],
                      ),
                      child: Row(
                        children: [
                          Padding(
                            padding: const EdgeInsets.only(left: 5.0),
                            child: IconButton(
                              icon: const Icon(Icons.add_circle, color: Colors.grey, size: 28),
                              onPressed: _isLoading ? null : _pickImage,
                            ),
                          ),
                          Expanded(
                            child: TextField(
                              controller: _messageController,
                              onSubmitted: (_) => _sendMessage(),
                              decoration: const InputDecoration(
                                hintText: "Pose ta question ici",
                                hintStyle: TextStyle(color: Color(0xFFBDBDBD), fontSize: 16),
                                border: InputBorder.none,
                                contentPadding: EdgeInsets.symmetric(vertical: 15),
                              ),
                            ),
                          ),
                          Padding(
                            padding: const EdgeInsets.only(right: 8.0),
                            child: GestureDetector(
                              onTap: _isLoading ? null : _sendMessage,
                              child: Container(
                                padding: const EdgeInsets.all(10),
                                decoration: BoxDecoration(
                                  color: _isLoading ? Colors.grey : Colors.red.shade200,
                                  shape: BoxShape.circle,
                                ),
                                child: _isLoading 
                                    ? const SizedBox(width: 20, height: 20, child: CircularProgressIndicator(color: Colors.white, strokeWidth: 2))
                                    : const Icon(Icons.arrow_upward, color: Colors.white, size: 20),
                              ),
                            ),
                          ),
                        ],
                      ),
                    ),
                  ],
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }

  Widget _buildWelcomeScreen() {
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        Container(
          width: 100,
          height: 100,
          decoration: BoxDecoration(
            shape: BoxShape.circle,
            gradient: const SweepGradient(colors: [Colors.blue, Colors.purple, Colors.red, Colors.orange, Colors.blue]),
            boxShadow: [BoxShadow(color: Colors.purple.withValues(alpha: 0.3), blurRadius: 20, spreadRadius: 5)],
          ),
          child: Padding(
            padding: const EdgeInsets.all(12.0),
            child: Container(decoration: const BoxDecoration(color: Colors.white, shape: BoxShape.circle)),
          ),
        ),
        const SizedBox(height: 40),
        // 🟢 ON AFFICHE LE VRAI NOM DE L'UTILISATEUR
        Text("Salut $_prenomUtilisateur !", style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 18)),
        const SizedBox(height: 5),
        const Text("Je suis ton assistant 🤖", style: TextStyle(fontSize: 16, fontWeight: FontWeight.w500)),
        const SizedBox(height: 25),
        Padding(
          padding: const EdgeInsets.symmetric(horizontal: 30.0),
          child: Text(
            "Posez moi la question que vous voulez savoir en\n${widget.nomMatiere}",
            textAlign: TextAlign.center,
            style: const TextStyle(fontSize: 15, fontWeight: FontWeight.w600),
          ),
        ),
      ],
    );
  }

  Widget _buildChatList() {
    return ListView.builder(
      controller: _scrollController,
      padding: const EdgeInsets.symmetric(horizontal: 15, vertical: 20),
      itemCount: _messages.length,
      itemBuilder: (context, index) {
        final msg = _messages[index];
        final isUser = msg['role'] == 'user';
        final hasImage = msg['image'] != null;
        
        if (isUser) {
          return Align(
            alignment: Alignment.centerRight,
            child: Container(
              margin: const EdgeInsets.only(bottom: 15, left: 40),
              padding: const EdgeInsets.all(12),
              decoration: BoxDecoration(
                color: Colors.red.shade400,
                borderRadius: const BorderRadius.only(
                  topLeft: Radius.circular(20),
                  topRight: Radius.circular(20),
                  bottomLeft: Radius.circular(20),
                  bottomRight: Radius.circular(0), 
                ),
                boxShadow: [BoxShadow(color: Colors.black.withValues(alpha: 0.5), blurRadius: 5, offset: const Offset(0, 2))],
              ),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.end,
                children: [
                  if (hasImage)
                    Padding(
                      padding: const EdgeInsets.only(bottom: 8.0),
                      child: ClipRRect(
                        borderRadius: BorderRadius.circular(10),
                        child: Image.file(msg['image'] as File, width: 200, fit: BoxFit.cover),
                      ),
                    ),
                  if (msg['text'] != null && msg['text'].toString().isNotEmpty)
                    Text(
                      msg['text'] ?? '',
                      style: const TextStyle(color: Colors.white, fontSize: 15, height: 1.4),
                    ),
                ],
              ),
            ),
          );
        } else {
          return Align(
            alignment: Alignment.centerLeft,
            child: Container(
              margin: const EdgeInsets.only(bottom: 25, right: 10, left: 5),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  if (msg['text'] != null && msg['text'].toString().isNotEmpty)
                    MarkdownBody(
                      data: msg['text'] ?? '',
                      selectable: true, 
                      builders: {
                        'latex': LatexElementBuilder(
                          textStyle: const TextStyle(color: Colors.black87, fontWeight: FontWeight.w500),
                        ),
                      },
                      extensionSet: md.ExtensionSet(
                        [LatexBlockSyntax(), ...md.ExtensionSet.gitHubFlavored.blockSyntaxes],
                        [LatexInlineSyntax(), ...md.ExtensionSet.gitHubFlavored.inlineSyntaxes],
                      ),
                      styleSheet: MarkdownStyleSheet(
                        p: const TextStyle(fontSize: 15, height: 1.6, color: Colors.black87),
                        h1: const TextStyle(fontSize: 22, fontWeight: FontWeight.w900, color: Colors.red),
                        h2: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold, color: Colors.black),
                        h3: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold, color: Colors.black87),
                        tableBorder: TableBorder.all(color: Colors.grey.shade300, width: 1),
                        tableCellsPadding: const EdgeInsets.all(12),
                        tableHead: const TextStyle(fontWeight: FontWeight.w900, color: Colors.black),
                        tableBody: const TextStyle(fontSize: 14, color: Colors.black87),
                        blockquoteDecoration: BoxDecoration(
                          color: Colors.red.shade50,
                          border: const Border(left: BorderSide(color: Colors.red, width: 4)),
                        ),
                        listBullet: const TextStyle(color: Colors.red, fontWeight: FontWeight.bold),
                      ),
                    ),
                ],
              ),
            ),
          );
        }
      },
    );
  }
}
