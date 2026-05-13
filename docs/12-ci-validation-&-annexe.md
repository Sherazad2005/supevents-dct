# CI Validation — TP 1.7

## Objectif
Vérifier que la CI fonctionne correctement sur les fichiers Markdown.

---

## Test réalisé

- Ajout d’un lien cassé dans un fichier `.md`
- Push dans une pull request
- CI déclenchée automatiquement

---

## Résultat

- ❌ CI en échec (liens invalides détectés)
- Correction du lien
- ✔ CI repasse au vert

---

## Preuve

Lien PR : [ajouter lien GitHub ici]

Capture CI : [optionnel]

---

## Conclusion

La pipeline CI valide correctement :
- lint Markdown
- validation des liens
- structure du workflow
