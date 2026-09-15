<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Importateur Intelligent de Contacts</title>
    <!-- Chargement de l'API Grist et de PapaParse pour lire les CSV proprement -->
    <script src="https://docs.getgrist.com/grist-plugin-api.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.4.1/papaparse.min.js"></script>
    <style>
        :root {
            --primary: #4f46e5;
            --primary-hover: #4338ca;
            --bg-color: #f8fafc;
            --card-bg: #ffffff;
            --text-main: #1e293b;
            --text-muted: #64748b;
            --border: #cbd5e1;
            --error-bg: #ffeeec;
            --error-border: #ef4444;
            --success: #10b981;
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background-color: var(--bg-color);
            color: var(--text-main);
            margin: 0;
            padding: 20px;
            box-sizing: border-box;
        }

        .container {
            max-width: 900px;
            margin: 0 auto;
            background: var(--card-bg);
            padding: 24px;
            border-radius: 12px;
            box-shadow: 0 4px 6px -1px rgb(0 0 0 / 0.1);
        }

        h2 {
            margin-top: 0;
            color: var(--text-main);
            font-size: 1.25rem;
            border-bottom: 2px solid #f1f5f9;
            padding-bottom: 12px;
        }

        /* Zone Dropzone */
        .dropzone {
            border: 2px dashed var(--border);
            border-radius: 8px;
            padding: 40px 20px;
            text-align: center;
            background: #fafafa;
            cursor: pointer;
            transition: all 0.2s ease;
            margin-bottom: 20px;
        }

        .dropzone.dragover {
            border-color: var(--primary);
            background: #eef2ff;
        }

        .dropzone p {
            margin: 10px 0;
            color: var(--text-muted);
        }

        .btn {
            background-color: var(--primary);
            color: white;
            border: none;
            padding: 10px 20px;
            font-size: 0.95rem;
            font-weight: 600;
            border-radius: 6px;
            cursor: pointer;
            transition: background 0.2s;
        }

        .btn:hover {
            background-color: var(--primary-hover);
        }

        .btn:disabled {
            background-color: var(--border);
            cursor: not-allowed;
        }

        /* Section de Mapping */
        #mapping-section {
            display: none;
            margin-top: 20px;
        }

        .mapping-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
            gap: 15px;
            max-height: 400px;
            overflow-y: auto;
            padding: 5px;
            margin-bottom: 20px;
        }

        .mapping-card {
            background: #fff;
            border: 1px solid var(--border);
            border-radius: 6px;
            padding: 12px;
            box-shadow: 0 1px 2px rgb(0 0 0 / 0.05);
        }

        .mapping-card.unrecognized {
            border-color: var(--error-border);
            background-color: var(--error-bg);
        }

        .mapping-card label {
            display: block;
            font-size: 0.85rem;
            color: var(--text-muted);
            margin-bottom: 4px;
        }

        .mapping-card .original-name {
            font-weight: bold;
            font-size: 0.95rem;
            margin-bottom: 8px;
            word-break: break-all;
        }

        .mapping-card select {
            width: 100%;
            padding: 6px;
            border: 1px solid var(--border);
            border-radius: 4px;
            font-size: 0.9rem;
            background: white;
        }

        .actions-bar {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-top: 20px;
            border-top: 1px solid #f1f5f9;
            padding-top: 15px;
        }

        #status-msg {
            font-weight: 500;
            font-size: 0.9rem;
        }
    </style>
</head>
<body>

<div class="container">
    <h2>Importer un fichier de contacts (.csv)</h2>

    <!-- Zone Glisser-Déposer -->
    <div id="dropzone" class="dropzone">
        <svg width="48" height="48" fill="none" stroke="currentColor" stroke-width="1.5" viewBox="0 0 24 24" style="color: var(--text-muted); margin-bottom: 8px;">
            <path stroke-linecap="round" stroke-linejoin="round" d="M3 16.5v2.25A2.25 2.25 0 005.25 21h13.5A2.25 2.25 0 0021 18.75V16.5m-13.5-9L12 3m0 0l4.5 4.5M12 3v13.5"></path>
        </svg>
        <p>Glissez votre fichier CSV ici ou</p>
        <button type="button" class="btn" onclick="document.getElementById('fileInput').click()">Parcourir les fichiers</button>
        <input type="file" id="fileInput" accept=".csv" style="display: none;">
    </div>

    <!-- Section de correspondance des colonnes (Mapping) -->
    <div id="mapping-section">
        <p style="margin-top: 0; color: var(--text-muted); font-size: 0.9rem;">
            Vérifiez l'association des colonnes de votre fichier avec vos champs Grist. Les éléments en <strong>rouge</strong> n'ont pas pu être reconnus automatiquement : veuillez leur assigner la bonne colonne cible.
        </p>
        
        <div id="mappingGrid" class="mapping-grid"></div>

        <div class="actions-bar">
            <span id="status-msg"></span>
            <button id="btnImport" class="btn" onclick="executeImport()">Lancer l'intégration</button>
        </div>
    </div>
</div>

<script>
    // Liste officielle de vos colonnes cibles dans la table CONTACTS de Grist
    const GRIST_COLUMNS = [
        "Cree_Par", "Date_Saisie", "Service", "DOMAINE D'ACTIVITES", 
        "Commentaire / Description", "STATUT JURIDIQUE", "TYPE_DE_STRUCTURE", 
        "NOM_Structure", "NOM", "PRENOM", "EMAIL PRINCIPAL", "EMAILS SECONDAIRES", 
        "Téléphone Principal", "Téléphones Secondaires", "Fonction / Profession", 
        "Code Postal", "Ville", "Adresse", "Pays", 
        "Listes_Diffusion_Intervention", "Listes_Diffusion_Cinema", 
        "Listes_Diffusion_Direction", "Listes_Diffusion_Houdremont"
    ];

    // Dictionnaire de synonymes pour la reconnaissance automatique intelligente
    const SYNONYMS = {
        "email principal": ["email", "mail", "courriel", "e-mail", "adresse email", "email principal"],
        "emails secondaires": ["emails secondaires", "autre mail", "mails secondaires", "second email"],
        "nom": ["nom", "lastname", "patronyme"],
        "prenom": ["prenom", "firstname", "prénom"],
        "nom_structure": ["nom structure", "structure", "entreprise", "organisme", "societe", "nom_structure"],
        "téléphone principal": ["telephone", "tel", "téléphone", "phone", "portable", "téléphone principal"],
        "code postal": ["code postal", "cp", "postal"],
        "ville": ["ville", "commune", "city"],
        "adresse": ["adresse", "rue", "address"],
        "pays": ["pays", "country"]
    };

    let parsedData = [];
    let fileHeaders = [];

    // Initialisation Grist
    grist.ready({ requiredAccess: 'full' });

    // Gestion du Drag & Drop
    const dropzone = document.getElementById('dropzone');
    const fileInput = document.getElementById('fileInput');

    ['dragenter', 'dragover'].forEach(eventName => {
        dropzone.addEventListener(eventName, (e) => { e.preventDefault(); dropzone.classList.add('dragover'); }, false);
    });
    ['dragleave', 'drop'].forEach(eventName => {
        dropzone.addEventListener(eventName, (e) => { e.preventDefault(); dropzone.classList.remove('dragover'); }, false);
    });

    dropzone.addEventListener('drop', (e) => {
        const dt = e.dataTransfer;
        const files = dt.files;
        if (files.length) handleFile(files[0]);
    });

    fileInput.addEventListener('change', (e) => {
        if (e.target.files.length) handleFile(e.target.files[0]);
    });

    function handleFile(file) {
        Papa.parse(file, {
            header: true,
            skipEmptyLines: true,
            complete: function(results) {
                if (results.data && results.data.length > 0) {
                    fileHeaders = results.meta.fields;
                    parsedData = results.data;
                    showMappingUI(fileHeaders);
                } else {
                    alert("Le fichier CSV est vide ou mal formaté.");
                }
            },
            error: function(err) {
                alert("Erreur lors de la lecture du fichier : " + err.message);
            }
        });
    }

    function findBestMatch(header) {
        const cleanHeader = header.trim().toLowerCase();
        // 1. Correspondance exacte insensible à la casse
        const exactMatch = GRIST_COLUMNS.find(col => col.toLowerCase() === cleanHeader);
        if (exactMatch) return exactMatch;

        // 2. Recherche par synonymes
        for (const [targetCol, keywords] of Object.entries(SYNONYMS)) {
            if (keywords.includes(cleanHeader)) {
                return GRIST_COLUMNS.find(col => col.toLowerCase() === targetCol.toLowerCase()) || "";
            }
        }
        return ""; // Non trouvé
    }

    function showMappingUI(headers) {
        const grid = document.getElementById('mappingGrid');
        grid.innerHTML = "";

        headers.forEach((header, index) => {
            const bestMatch = findBestMatch(header);
            const isRecognized = bestMatch !== "";

            const card = document.createElement('div');
            card.className = `mapping-card ${isRecognized ? '' : 'unrecognized'}`;
            card.dataset.original = header;

            let optionsHtml = `<option value="">-- Ignorer / Non assigné --</option>`;
            GRIST_COLUMNS.forEach(col => {
                const selected = (col === bestMatch) ? 'selected' : '';
                optionsHtml += `<option value="${col}" ${selected}>${col}</option>`;
            });

            card.innerHTML = `
                <label>Colonne du CSV :</label>
                <div class="original-name">${header}</div>
                <label>Correspondance Grist :</label>
                <select class="mapping-select" data-index="${index}">
                    ${optionsHtml}
                </select>
            `;
            grid.appendChild(card);
        });

        document.getElementById('mapping-section').style.display = 'block';
        document.getElementById('status-msg').innerText = `${parsedData.length} lignes prêtes à être analysées.`;
    }

    async function executeImport() {
        const selects = document.querySelectorAll('.mapping-select');
        const mapping = {};
        
        selects.forEach(select => {
            const originalHeader = select.closest('.mapping-card').dataset.original;
            const targetCol = select.value;
            if (targetCol) {
                mapping[originalHeader] = targetCol;
            }
        });

        if (Object.keys(mapping).length === 0) {
            alert("Veuillez associer au moins une colonne avant d'importer.");
            return;
        }

        const btn = document.getElementById('btnImport');
        btn.disabled = true;
        document.getElementById('status-msg').innerText = "Importation en cours...";

        // Transformation des données du CSV selon le mapping choisi
        const recordsToInsert = parsedData.map(row => {
            const newRecord = {};
            for (const [origKey, targetKey] of Object.entries(mapping)) {
                if (row[origKey] !== undefined) {
                    newRecord[targetKey] = row[origKey];
                }
            }
            return newRecord;
        });

        try {
            // Insertion directe dans la table active de Grist
            await grist.selectedTable.create(recordsToInsert);
            document.getElementById('status-msg').innerText = `Succès ! ${recordsToInsert.length} contacts ont été importés.`;
            document.getElementById('status-msg').style.color = "var(--success)";
            setTimeout(() => {
                location.reload(); // Réinitialise l'outil pour un nouvel import
            }, 2500);
        } catch (err) {
            alert("Erreur lors de l'enregistrement dans Grist : " + err.message);
            btn.disabled = false;
            document.getElementById('status-msg').innerText = "Échec de l'importation.";
        }
    }
</script>

</body>
</html>
