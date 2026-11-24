# Guidelines — Portfolio « Maison de poupée »

> Document de référence. Toute modification du site doit respecter ces règles.
> Les points marqués **[À DÉFINIR]** sont des décisions ouvertes : ne pas les inventer, demander.

## 1. Vision
Le portfolio est une maison de poupée ouverte. Le visiteur arrive devant une façade close,
la maison s'ouvre, puis il explore les pièces en cliquant dessus : la caméra zoome dans la
pièce, où des objets 3D révèlent le contenu. Sortir d'une pièce doit toujours être immédiat.
Ton : tactile, curieux, « collage » de matières (léopard, mosaïque miroir, parquet, damier).

## 2. Source de design
- Fichier Figma : `NoBIkiXvccvt8FaQJBlNg5`, node `23:156` « Dollhouse / 9 connected rooms ».
- Composants pièces Figma : `Room / Rose tile with opening` (13:2), `Room / Mirror mosaic` (13:3),
  `Room / Inlaid parquet` (13:4), `Room / Checkerboard stone` (13:5), + variantes carpet.
- Chaque pièce = 5 surfaces : Ceiling, Left wall, Right wall, Back wall, Floor (trapèzes en
  perspective). On reproduit cette structure en un seul composant `<Room>` piloté par des données.
- Textures exportées dans `public/assets/textures/` (WebP/AVIF), jamais d'URL Figma temporaire.

## 3. Plan de la maison
Grille (ligne, colonne), du haut vers le bas :

| Pos | Ambiance Figma                     | Rôle         | Route          |
|-----|------------------------------------|--------------|----------------|
| 1,1 | Murs léopard, fenêtre ciel, sol rose | **About**    | `/:lang/about` |
| 1,2 | Sol fleurs rouges, mur droit vert  | **Projects** | `/:lang/projects` (+ `/:lang/projects/:slug`) |
| 1,3 | Sol mosaïque miroir                | **Playground** | `/:lang/playground` |
| 2,1 | Sol écailles                       | Déco         | — |
| 2,2 | Parquet marqueté, mur miroir       | Déco         | — |
| 2,3 | Tapis fleurs rouges ASCII          | Déco         | — |
| 3,1 | Tapis abstrait                     | Déco         | — |
| 3,2 | Tapis géométrique                  | Déco         | — |
| 3,3 | Casiers + damier                   | **Contact**  | `/:lang/contact` |

La source de vérité est `src/content/rooms.ts` (id, position, rôle, textures, objets, route).
Aucune pièce n'est codée en dur ailleurs.

### Objets par pièce (proposition, **[À VALIDER]**)
- About : portrait encadré (bio), valise (parcours), plante (centres d'intérêt).
- Projects : un objet par projet phare (3 max visibles), ex. livres / maquettes sur étagère.
- Playground : boule à facettes / jouets → expérimentations.
- Contact : casier ouvert (email), téléphone (réseaux), sac (CV à télécharger).
Maximum 3 objets cliquables par pièce.

### Pièces décoratives
- Jour : statiques, léger effet au survol (lumière plus chaude), non cliquables, pas de zoom.
- Nuit : voir §6. Contenu futur **[À DÉFINIR]** — chaque pièce déco expose un slot `nightContent`.

## 4. Parcours & états
Machine d'états unique (`src/state/houseMachine.ts`) :
`loading → opening → house → zooming-in → room → closeup → room → zooming-out → house`

1. **loading** — Façade fermée (porte, fenêtres, toit), barre/jauge de chargement discrète
   intégrée au décor (ex. lumière qui s'allume derrière la porte). Progression réelle :
   textures de la maison + polices. Durée min. 1,2 s ; ignorée (ouverture directe) si déjà
   vue dans la session (`sessionStorage`).
2. **opening** — La façade pivote sur une charnière gauche (`rotateY` 0 → -105°,
   `perspective` ~1600px, ~1,4 s, easing doux), révélant la grille des pièces.
3. **house** — Vue d'ensemble. Pièces actives : affordance claire (léger zoom + étiquette au
   survol/focus ; sur mobile, étiquettes toujours visibles).
4. **zooming-in** — La « caméra » (un transform `translate + scale` sur le conteneur de la
   maison, calculé depuis le rect de la pièce) cadre la pièce plein écran (~700 ms).
   L'URL change au début du zoom.
5. **room** — La pièce occupe le viewport ; le canvas 3D des objets s'y superpose.
6. **closeup** — Clic sur un objet : l'objet s'avance / s'agrandit, puis le contenu lisible
   s'affiche (carte thématique). Desktop : carte latérale ; mobile : feuille plein écran depuis le bas.

### Sortie (toujours au moins 3 moyens)
- Bouton « ← Maison » fixe en haut à gauche (cible ≥ 44px).
- Touche `Échap` / bouton retour navigateur (l'historique suit les états).
- Mobile : swipe vers le bas sur le gros plan ; tap hors de la pièce.
Depuis un gros plan, la sortie ramène à la pièce ; depuis la pièce, à la maison.

### Navigation secondaire
Un petit « plan de la maison » (menu) permet d'aller directement à une pièce active
sans repasser par la vue d'ensemble (zoom-out → zoom-in enchaînés).

## 5. Architecture technique
- Stack : React 19 + Vite + Tailwind v4 (existant).
- Ajouts : `react-router` (routes), `motion` (caméra, façade, transitions UI),
  `three` + `@react-three/fiber` + `@react-three/drei` (objets 3D uniquement).
- Murs, sols, plafonds : **CSS/SVG 2.5D**, jamais en WebGL.
- **Un seul `<Canvas>`** R3F, monté uniquement dans les états `room`/`closeup`, chargé en lazy
  (`React.lazy`) : three.js n'est jamais dans le bundle initial.
- Dans la vue d'ensemble, les objets sont représentés par des **posters PNG/WebP** pré-rendus
  (même cadrage que la 3D) → la maison reste légère.
- Routes : `/` → redirection vers `/fr` ou `/en` selon `navigator.language` ;
  `/:lang` = maison ; `/:lang/:room` = pièce ; `/:lang/:room/:object` = gros plan.
  Arriver directement sur une URL de pièce : courte ouverture, puis zoom direct.
- i18n : dictionnaires `src/content/i18n/fr.ts` et `en.ts`, typés ; sélecteur FR/EN visible.
  Aucune chaîne en dur dans les composants.

### Arborescence cible
```
src/
  app/            routes, layout, providers (lang, theme jour/nuit)
  house/          House, Facade, Roof, RoomGrid, Camera (zoom), Loader
  room/           Room (5 surfaces), RoomView, ExitButton, Closeup
  three/          ObjectsCanvas, Model (useGLTF), lights, poster fallback
  content/        rooms.ts, projects.ts, i18n/{fr,en}.ts
  state/          houseMachine.ts
  styles/         tokens dans src/index.css (@theme)
public/assets/
  textures/       murs, sols (webp/avif, 1x/2x)
  models/         *.glb compressés
  posters/        rendus 2D des objets
```

## 6. Mode jour / nuit
- Interrupteur visible sur la maison (ex. une lampe ou un interrupteur sur la façade),
  préférence initiale = `prefers-color-scheme`, mémorisée dans `localStorage`.
- Nuit : filtre/assombrissement des surfaces 2D, fenêtres et pièces actives éclairées
  (halo chaud), lumières 3D ajustées dans le canvas.
- Pièces décoratives : de nuit, elles révèlent leur slot `nightContent` ; tant qu'il est vide,
  afficher une ambiance (silhouette, lumière tamisée) **[contenu À DÉFINIR]**.
- Implémenté via une classe `.night` sur `<html>` + variables CSS ; pas de duplication de composants.

## 7. Mobile d'abord
- Le design est portrait : sur mobile, la maison tient **entière** dans `100dvh`
  (toit compris), conteneur à ratio fixe, mis à l'échelle par
  `min(100vw, 100dvh × ratio)`. Sur desktop, maison centrée avec espace décor autour.
- Cibles tactiles : une pièce ≈ 1/3 de la largeur (≥ 110px sur 360px) — OK ; boutons ≥ 44px.
- Pas de survol sur mobile : étiquettes des pièces actives toujours visibles, pulsation légère
  une fois après l'ouverture pour signaler l'interactivité.
- En pièce, sur mobile : la pièce est cadrée en « cover » (on peut rogner le plafond),
  objets placés dans la moitié basse/centrale (zone du pouce).
- Gros plan mobile = feuille plein écran, scroll interne, fermable par swipe bas.
- Pas de pinch-zoom personnalisé ni de scroll-jacking.
- Gestes : tap uniquement pour entrer ; les objets 3D peuvent tourner au drag (OrbitControls
  limités) mais jamais bloquer le scroll de la feuille.

## 8. Performance (budgets)
- JS initial (sans three) ≤ 180 KB gzip ; chunk 3D chargé à l'entrée d'une pièce,
  préchargé au survol/focus d'une pièce active et en idle après l'ouverture.
- Textures : WebP/AVIF, ≤ 150 KB chacune, variantes 1x/2x ; maison complète ≤ 1,5 MB.
- Modèles : `.glb` compressés (Draco ou Meshopt via gltf-transform), ≤ 500 KB par pièce,
  textures ≤ 1024px, ≤ 3 modèles par pièce.
- Canvas : `dpr={[1, 1.75]}`, `frameloop="demand"`, pas d'ombres temps réel sur mobile
  (ombres pré-cuites ou contact shadows).
- Animations uniquement sur `transform`/`opacity` ; 60 fps visé sur mobile milieu de gamme.

## 9. Accessibilité & résilience
- Chaque pièce active est un lien (`<a>`) avec libellé ; chaque objet 3D a un bouton HTML
  équivalent (overlay invisible positionné ou liste) → navigable au clavier.
- Focus géré : focus sur le titre de la pièce à l'entrée, piège de focus dans le gros plan,
  retour du focus sur la pièce/objet d'origine à la sortie.
- `prefers-reduced-motion` : façade en fondu, zoom remplacé par fondu court, pas d'auto-rotation.
- Pas de WebGL / échec de chargement : on affiche les posters 2D cliquables, même parcours.
- Contraste AA sur tout texte ; aucun contenu uniquement dans la 3D.
- SEO : titre et meta par route et par langue ; contenu textuel rendu en HTML.

## 10. Direction visuelle
- Fidélité au Figma pour la maison : contours fins sombres (~#2b2b2b), murs gris clair,
  fond de pièce sombre, textures collage.
- Les objets 3D doivent coller au style collage : modèles réalistes ou photogrammétrie
  légère, pas de low-poly « jouet » généré en code.
- Typographie **[À DÉFINIR]** (suggestion : une serif éditoriale pour les titres + une sans
  ou mono pour les étiquettes façon « musée »).
- Pas d'emoji dans l'interface ; icônes d'un seul set cohérent.

## 11. Sources des modèles 3D
- Autorisées : modèles fournis par l'autrice (.glb), bibliothèques libres (Poly Pizza,
  Sketchfab CC0/CC-BY, Poly Haven).
- Chaque modèle est listé dans `public/assets/models/CREDITS.md` (source, auteur, licence).
  Les licences CC-BY sont créditées dans le footer / page Contact.
- En attendant un modèle : poster 2D comme placeholder, jamais un mesh inventé.

## 12. Contenu
- Textes, projets, liens : `src/content/` (pas de CMS en v1).
- Projet = `{ slug, title{fr,en}, summary{fr,en}, year, role, media[], links[], object }`.
- **[À DÉFINIR]** : liste des projets, textes About, canaux de contact, CV.

## 13. Ordre de construction recommandé
1. Tokens + maison statique (grille 3×3, `<Room>` data-driven, textures) responsive.
2. Routing + caméra zoom in/out + sorties (sans 3D, avec posters).
3. Loader + façade qui pivote ; mode reduced-motion.
4. Canvas 3D lazy + 1 objet dans About → valider perf mobile réelle.
5. Gros plans + contenu des 4 pièces, FR/EN.
6. Mode nuit + pièces décoratives.
7. Passe a11y, SEO, budgets perf.

## 14. Définition de « terminé » pour chaque étape
- Testé à 360×740, 390×844, 768×1024, 1440×900.
- Entrer/sortir d'une pièce au tap, au clavier, et via le bouton retour navigateur.
- Reduced-motion et absence de WebGL vérifiés.
- Aucun asset Figma temporaire référencé ; budgets §8 respectés.
