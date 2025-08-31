<?php
// ==============================================================================
// 1. GESTION DU BACK-END (PHP)
//    Ce script gère les requêtes vers la base de données.
// ==============================================================================

// Nom du fichier de base de données SQLite.
// KSWEB supporte SQLite, ce qui est idéal pour une utilisation locale.
$databaseFile = 'vote.db';

// Créer la base de données et les tables si elles n'existent pas.
try {
    $db = new PDO("sqlite:$databaseFile");
    $db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

    // Table des étudiants
    $db->exec("CREATE TABLE IF NOT EXISTS etudiants (
        code_etudiant TEXT PRIMARY KEY,
        nom_complet TEXT,
        faculte TEXT,
        promotion INTEGER,
        a_vote INTEGER DEFAULT 0
    )");

    // Table des votes
    $db->exec("CREATE TABLE IF NOT EXISTS votes (
        id_vote INTEGER PRIMARY KEY AUTOINCREMENT,
        code_etudiant TEXT,
        id_candidat INTEGER,
        date_vote TEXT,
        FOREIGN KEY (code_etudiant) REFERENCES etudiants(code_etudiant)
    )");
    
    // Table des candidats
    $db->exec("CREATE TABLE IF NOT EXISTS candidats (
        id_candidat INTEGER PRIMARY KEY,
        nom TEXT,
        position TEXT
    )");
    
    // --- Insérer des données de test si les tables sont vides ---
    $etudiantsCount = $db->query("SELECT count(*) FROM etudiants")->fetchColumn();
    if ($etudiantsCount == 0) {
        $stmt = $db->prepare("INSERT INTO etudiants (code_etudiant, nom_complet, faculte, promotion) VALUES (?, ?, ?, ?)");
        
        // Données des étudiants
        for ($i = 1001; $i <= 1050; $i++) {
            $stmt->execute(["25informatique$i", "Étudiant $i", "Informatique", 25]);
        }
        // Ajoutez d'autres facultés/promotions si nécessaire
        for ($i = 1001; $i <= 1020; $i++) {
            $stmt->execute(["25genie$i", "Étudiant GC $i", "Génie Civil", 25]);
        }
    }

    $candidatsCount = $db->query("SELECT count(*) FROM candidats")->fetchColumn();
    if ($candidatsCount == 0) {
        $stmt = $db->prepare("INSERT INTO candidats (id_candidat, nom, position) VALUES (?, ?, ?)");
        $stmt->execute([1, "Jean Kabila", "Président AEUM"]);
        $stmt->execute([2, "Marie Tshibangu", "Vice-Président AEUM"]);
        $stmt->execute([3, "Paul Mutombo", "Secrétaire AEUM"]);
        $stmt->execute([4, "Sarah Kabeya", "Trésorier AEUM"]);
    }
    
} catch (PDOException $e) {
    die("Erreur de connexion à la base de données : " . $e->getMessage());
}

// Gérer les requêtes POST (login et soumission de vote)
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    header('Content-Type: application/json');
    $data = json_decode(file_get_contents('php://input'), true);
    
    $action = $data['action'] ?? '';
    
    if ($action === 'login') {
        $code = $data['code'] ?? '';
        $stmt = $db->prepare("SELECT a_vote FROM etudiants WHERE code_etudiant = ?");
        $stmt->execute([$code]);
        $result = $stmt->fetch(PDO::FETCH_ASSOC);
        
        if ($result && $result['a_vote'] == 0) {
            echo json_encode(["success" => true]);
        } else {
            echo json_encode(["success" => false, "message" => "Code invalide ou déjà utilisé."]);
        }
        exit;
        
    } elseif ($action === 'submit_vote') {
        $code = $data['code'] ?? '';
        $candidateId = $data['candidateId'] ?? '';
        
        $db->beginTransaction();
        try {
            // Vérifier si l'étudiant n'a pas déjà voté
            $stmt = $db->prepare("SELECT a_vote FROM etudiants WHERE code_etudiant = ?");
            $stmt->execute([$code]);
            $etudiant = $stmt->fetch(PDO::FETCH_ASSOC);
            
            if (!$etudiant || $etudiant['a_vote'] == 1) {
                $db->rollBack();
                echo json_encode(["success" => false, "message" => "Vote non autorisé."]);
                exit;
            }
            
            // Enregistrer le vote
            $stmt = $db->prepare("INSERT INTO votes (code_etudiant, id_candidat, date_vote) VALUES (?, ?, ?)");
            $stmt->execute([$code, $candidateId, date('Y-m-d H:i:s')]);
            
            // Marquer l'étudiant comme ayant voté
            $stmt = $db->prepare("UPDATE etudiants SET a_vote = 1 WHERE code_etudiant = ?");
            $stmt->execute([$code]);
            
            $db->commit();
            
            // Générer un numéro de confirmation aléatoire
            $confirmation = 'UM-' . str_pad(mt_rand(1, 999999), 6, '0', STR_PAD_LEFT);
            echo json_encode(["success" => true, "confirmation_number" => $confirmation]);
            
        } catch (Exception $e) {
            $db->rollBack();
            echo json_encode(["success" => false, "message" => "Erreur lors de l'enregistrement du vote."]);
        }
        exit;
    }
}
?>

<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Système de Vote en Ligne - Université de Tshikama</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }
        
        body {
            background-color: #f5f5f5;
            color: #333;
            line-height: 1.6;
        }
        
        .container {
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
        }
        
        header {
            background-color: #1a237e;
            color: white;
            padding: 20px 0;
            text-align: center;
            border-radius: 5px 5px 0 0;
            margin-bottom: 20px;
        }
        
        .logo {
            font-size: 24px;
            font-weight: bold;
            margin-bottom: 10px;
        }
        
        h1 {
            font-size: 22px;
            margin-bottom: 10px;
        }
        
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        
        .form-group {
            margin-bottom: 15px;
        }
        
        label {
            display: block;
            margin-bottom: 5px;
            font-weight: 600;
        }
        
        input[type="text"] {
            width: 100%;
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 4px;
            font-size: 16px;
        }
        
        button {
            background-color: #1a237e;
            color: white;
            border: none;
            padding: 12px 20px;
            border-radius: 4px;
            cursor: pointer;
            font-size: 16px;
            font-weight: 600;
            transition: background-color 0.3s;
        }
        
        button:hover {
            background-color: #303f9f;
        }
        
        .candidates {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
            gap: 15px;
            margin-top: 20px;
        }
        
        .candidate {
            border: 1px solid #ddd;
            border-radius: 4px;
            padding: 15px;
            text-align: center;
            cursor: pointer;
            transition: all 0.3s;
        }
        
        .candidate:hover {
            background-color: #f0f0f0;
            transform: translateY(-2px);
        }
        
        .candidate.selected {
            border-color: #1a237e;
            background-color: #e8eaf6;
        }
        
        .hidden {
            display: none;
        }
        
        .message {
            padding: 15px;
            margin: 15px 0;
            border-radius: 4px;
            text-align: center;
        }
        
        .success {
            background-color: #e8f5e9;
            color: #2e7d32;
            border: 1px solid #c8e6c9;
        }
        
        .error {
            background-color: #ffebee;
            color: #c62828;
            border: 1px solid #ffcdd2;
        }
        
        footer {
            text-align: center;
            margin-top: 30px;
            color: #666;
            font-size: 14px;
        }
    </style>
</head>
<body>
    <div class="container">
        <header>
            <div class="logo">UT - Université de Tshikama</div>
            <h1>Système de Vote en Ligne pour les Élections Étudiantes</h1>
        </header>
        
        <div id="login-section" class="card">
            <h2>Identification de l'étudiant</h2>
            <p>Veuillez entrer votre nom et code étudiant pour accéder au système de vote.</p>
            
            <div class="form-group">
                <label for="name">Nom complet:</label>
                <input type="text" id="name" placeholder="Votre nom et prénom">
            </div>
            
            <div class="form-group">
                <label for="code">Code étudiant:</label>
                <input type="text" id="code" placeholder="Votre code étudiant (ex: 25informatique1001)">
            </div>
            
            <button id="login-btn">Voter</button>
            
            <div id="login-message" class="message hidden"></div>
        </div>
        
        <div id="voting-section" class="card hidden">
            <h2>Procédez à votre vote</h2>
            <p id="welcome-message">Bonjour [Nom], veuillez sélectionner votre candidat:</p>
            
            <div class="candidates" id="candidates-list">
                </div>
            
            <button id="submit-vote" disabled>Soumettre mon vote</button>
            <div id="vote-message" class="message hidden"></div>
        </div>
        
        <div id="confirmation-section" class="card hidden">
            <h2>Vote enregistré avec succès!</h2>
            <div class="message success">
                <p>Merci d'avoir participé aux élections de l'Université de Tshikama.</p>
                <p>Votre vote a été enregistré et ne pourra pas être modifié.</p>
            </div>
            <p>Numéro de confirmation: <span id="confirmation-number"></span></p>
        </div>
        
        <footer>
            <p>© 2025 Université de Tshikama - Tous droits réservés</p>
            <p>Système sécurisé de vote en ligne crée par: ir Kalambayi Kalala Manasse & ir Muipatayi Tshishimbi Fridolin</p>
        </footer>
    </div>

    <script>
        // ======================================================================
        // 3. LOGIQUE FRONT-END (JAVASCRIPT)
        //    Ce script gère l'interface et les interactions avec l'utilisateur.
        // ======================================================================

        // Données des candidats (chargées en dur pour la démo, pourraient être chargées depuis le serveur)
        const candidates = [
            { id: 1, name: "Jean Kabila", position: "Président AEUM" },
            { id: 2, name: "Marie Tshibangu", position: "Vice-Président AEUM" },
            { id: 3, name: "Paul Mutombo", position: "Secrétaire AEUM" },
            { id: 4, name: "Sarah Kabeya", position: "Trésorier AEUM" }
        ];

        // Références aux éléments DOM
        const loginSection = document.getElementById('login-section');
        const votingSection = document.getElementById('voting-section');
        const confirmationSection = document.getElementById('confirmation-section');
        const loginBtn = document.getElementById('login-btn');
        const nameInput = document.getElementById('name');
        const codeInput = document.getElementById('code');
        const loginMessage = document.getElementById('login-message');
        const welcomeMessage = document.getElementById('welcome-message');
        const candidatesList = document.getElementById('candidates-list');
        const submitVoteBtn = document.getElementById('submit-vote');
        const voteMessage = document.getElementById('vote-message');
        const confirmationNumber = document.getElementById('confirmation-number');
        
        // Variables d'état
        let selectedCandidate = null;
        let currentStudent = null;
        
        function initializeCandidates() {
            candidatesList.innerHTML = '';
            candidates.forEach(candidate => {
                const candidateElement = document.createElement('div');
                candidateElement.className = 'candidate';
                candidateElement.dataset.id = candidate.id;
                
                candidateElement.innerHTML = `
                    <h3>${candidate.name}</h3>
                    <p>${candidate.position}</p>
                `;
                
                candidateElement.addEventListener('click', () => {
                    document.querySelectorAll('.candidate').forEach(c => c.classList.remove('selected'));
                    candidateElement.classList.add('selected');
                    selectedCandidate = candidate.id;
                    submitVoteBtn.disabled = false;
                });
                
                candidatesList.appendChild(candidateElement);
            });
        }
        
        function initializeApp() {
            initializeCandidates();
            
            // Événement de connexion
            loginBtn.addEventListener('click', () => {
                const name = nameInput.value.trim();
                const code = codeInput.value.trim();
                
                if (!name || !code) {
                    loginMessage.textContent = 'Veuillez entrer votre nom et code étudiant.';
                    loginMessage.className = 'message error';
                    loginMessage.classList.remove('hidden');
                    return;
                }
                
                // Appel au serveur PHP pour la vérification
                fetch('vote.php', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ action: 'login', code: code })
                })
                .then(response => response.json())
                .then(data => {
                    if (data.success) {
                        currentStudent = { name, code };
                        welcomeMessage.textContent = `Bonjour ${name}, veuillez sélectionner votre candidat:`;
                        loginSection.classList.add('hidden');
                        votingSection.classList.remove('hidden');
                        loginMessage.classList.add('hidden');
                    } else {
                        loginMessage.textContent = data.message;
                        loginMessage.className = 'message error';
                        loginMessage.classList.remove('hidden');
                    }
                })
                .catch(error => {
                    console.error('Erreur:', error);
                    loginMessage.textContent = 'Erreur de connexion au serveur.';
                    loginMessage.className = 'message error';
                    loginMessage.classList.remove('hidden');
                });
            });
            
            // Événement de soumission du vote
            submitVoteBtn.addEventListener('click', () => {
                if (!selectedCandidate) {
                    voteMessage.textContent = 'Veuillez sélectionner un candidat.';
                    voteMessage.className = 'message error';
                    voteMessage.classList.remove('hidden');
                    return;
                }
                
                // Appel au serveur PHP pour enregistrer le vote
                fetch('vote.php', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        action: 'submit_vote',
                        code: currentStudent.code,
                        candidateId: selectedCandidate
                    })
                })
                .then(response => response.json())
                .then(data => {
                    if (data.success) {
                        confirmationNumber.textContent = data.confirmation_number;
                        votingSection.classList.add('hidden');
                        confirmationSection.classList.remove('hidden');
                    } else {
                        voteMessage.textContent = data.message;
                        voteMessage.className = 'message error';
                        voteMessage.classList.remove('hidden');
                    }
                })
                .catch(error => {
                    console.error('Erreur:', error);
                    voteMessage.textContent = 'Erreur lors de l\'envoi du vote.';
                    voteMessage.className = 'message error';
                    voteMessage.classList.remove('hidden');
                });
            });
        }
        
        document.addEventListener('DOMContentLoaded', initializeApp);
    </script>
</body>
</html>
