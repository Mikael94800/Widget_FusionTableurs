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
    document.getElementById('status-msg').innerText = "Préparation des données...";

    // Transformation des données du CSV
    const recordsToInsert = parsedData.map(row => {
        const newRecord = {};
        for (const [origKey, targetKey] of Object.entries(mapping)) {
            if (row[origKey] !== undefined && row[origKey] !== "") {
                newRecord[targetKey] = row[origKey];
            }
        }
        return newRecord;
    });

    console.log("Données prêtes à être envoyées à Grist :", recordsToInsert);

    try {
        document.getElementById('status-msg').innerText = "Envoi à Grist en cours...";
        // Insertion directe dans la table active de Grist
        const result = await grist.selectedTable.create(recordsToInsert);
        console.log("Réponse de Grist après insertion :", result);
        
        document.getElementById('status-msg').innerText = `Succès ! ${recordsToInsert.length} contacts ont été importés.`;
        document.getElementById('status-msg').style.color = "var(--success)";
        
        setTimeout(() => {
            location.reload();
        }, 2500);
    } catch (err) {
        console.error("Erreur détaillée Grist :", err);
        alert("Erreur lors de l'enregistrement dans Grist : " + (err.message || JSON.stringify(err)));
        btn.disabled = false;
        document.getElementById('status-msg').innerText = "Échec de l'importation.";
    }
}
