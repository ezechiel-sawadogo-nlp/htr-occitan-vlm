# HTR de manuscrits occitans médiévaux : Kraken vs VLM

🇬🇧 [English version](README.en.md)

Reconnaissance automatique de l'écriture manuscrite (HTR) sur des manuscrits **occitans médiévaux** : comparaison rigoureuse entre un modèle spécialisé léger (**Kraken**, 4 M de paramètres) et un modèle vision-langage généraliste (**Qwen2.5-VL-3B**, 3,8 Md de paramètres), avec et sans affinage, puis export des transcriptions en **XML-TEI**.

## Résultats

CER (taux d'erreur par caractère) sur 574 lignes de test issues de **3 manuscrits jamais vus à l'entraînement**.

| Modèle | Paramètres | CER global | additional-21218 | arsenal-6355 | naf-6195 |
|---|---|---|---|---|---|
| Kraken CATMuS Medieval, tel quel | 4 M | 16,7 % | 10,5 % | 7,7 % | 24,5 % |
| **Kraken CATMuS Medieval, affiné** | 4 M | **7,2 %** | **6,5 %** | **5,0 %** | **8,5 %** |
| Qwen2.5-VL-3B-Instruct, tel quel | 3,8 Md | 38,9 % | 33,8 % | 32,8 % | 44,9 % |
| Qwen2.5-VL-3B + LoRA | 3,8 Md | 22,1 % | 22,2 % | 17,8 % | 23,4 % |
| Qwen2.5-VL-3B + LoRA, images ×2 | 3,8 Md | 30,2 % | 29,9 % | 28,3 % | 31,1 % |
| **Qwen2.5-VL-3B + LoRA, langue + vision** | 3,8 Md | **7,9 %** | 9,1 % | 5,9 % | **7,6 %** |

**Conclusion principale** : un VLM généraliste n'atteint le niveau d'un modèle HTR spécialisé que si **son encodeur visuel est adapté**. Avec LoRA sur le seul modèle de langue, Qwen reste trois fois moins précis que Kraken affiné (22,1 % contre 7,2 %) ; avec LoRA aussi sur l'encodeur visuel, il tombe à **7,9 %**. L'écart restant avec Kraken (+0,7 point) **n'est pas statistiquement significatif** (intervalle de confiance à 95 % par bootstrap : [−0,4 ; +2,2] points), et Qwen fait mieux que Kraken sur le manuscrit aux lignes inclinées (`naf-6195`). À précision comparable, Kraken reste environ 900 fois plus léger et tourne sur CPU.

## Analyse d'erreurs

### Types d'erreurs (en proportion des caractères de référence)

| Modèle | CER | CER sans espaces | Substitutions | Suppressions | Insertions |
|---|---|---|---|---|---|
| Kraken tel quel | 16,7 % | 17,2 % | 7,2 % | 7,1 % | 2,4 % |
| Kraken affiné | 7,2 % | 7,1 % | 3,7 % | 2,1 % | 1,4 % |
| Qwen tel quel | 38,9 % | 42,1 % | 22,6 % | 7,8 % | 8,5 % |
| Qwen + LoRA | 22,1 % | 24,1 % | 13,5 % | 5,3 % | 3,4 % |
| Qwen + LoRA, images ×2 | 30,2 % | 33,0 % | 18,5 % | 7,6 % | 4,1 % |
| Qwen + LoRA langue + vision | 7,9 % | 7,9 % | 4,2 % | 2,0 % | 1,7 % |

### Part des caractères correctement lus, par catégorie

| Catégorie | Effectif | Kraken affiné | Qwen + LoRA | Qwen + LoRA, ×2 | Qwen + LoRA langue + vision |
|---|---|---|---|---|---|
| Lettres a-z | 12 693 | 94,4 % | 79,1 % | 70,6 % | 93,7 % |
| Espaces | 2 615 | 96,9 % | 94,0 % | 91,6 % | 97,4 % |
| Ponctuation / chiffres | 200 | 88,5 % | 88,5 % | 84,5 % | 87,5 % |
| Abréviations et signes spéciaux | 204 | 52,9 % | 45,6 % | 36,8 % | 57,8 % |

### Ce que montrent ces chiffres

1. **La segmentation des lignes compte autant que le modèle.** Découper chaque ligne selon son *rectangle* englobant incluait les lignes voisines sur les manuscrits photographiés de travers. Le passage à un découpage par *polygone* fait passer le CER de Kraken (tel quel) de **40,1 % à 16,7 %**, sans toucher au modèle.
2. **Kraken tel quel « oubliait » des caractères.** Ses suppressions (7,1 %) égalaient ses substitutions : 35 caractères du corpus occitan (chiffres, `k`, abréviations comme `ł` ou `ħ`, lettres suscrites) étaient absents de son alphabet. Après extension de l'alphabet et affinage, elles tombent à 2,1 %.
3. **L'affinage divise les hallucinations du VLM par 2,5.** Tel quel, Qwen « modernise » le texte : apostrophes, accents, et même des mots de français moderne absents de l'image (« Dauãt que 9dampnatz » → « D'aumône que la plume de la vaine »). Les insertions passent de 8,5 % à 3,4 %.
4. **Les espaces ne sont pas la principale source d'erreur** : retirer les espaces ne fait pas baisser le CER.
5. **L'écart entre Kraken et Qwen vient des lettres ordinaires**, pas des abréviations : 94,4 % contre 79,1 % de lettres a-z bien lues. Les abréviations sont difficiles pour les deux modèles, mais ne représentent que 1,3 % des caractères.
6. **Certaines confusions sont communes aux deux modèles** et tiennent à l'écriture elle-même : `l` ↔ `s` (le *s long* ressemble à un `l`), `u` ↔ `n` ↔ `m` (lettres faites des mêmes jambages).
7. **Une confusion est propre à Qwen : beaucoup de lettres deviennent `i`** (`a → i`, `u → i`, `r → i`, `y → i` ; 254 confusions vers `i`, contre 45 pour Kraken). Hypothèse testée : à la résolution native, une lettre occupe environ un seul token d'image (28 × 28 px), et le modèle percevrait des traits verticaux sans distinguer la forme des lettres.
8. **Agrandir les images ×2 ne résout pas le problème : le CER passe de 22,1 % à 30,2 %.** La prédiction posée avant l'expérience était que les confusions vers `i` baisseraient : elles augmentent (254 → 301), et de nouvelles confusions entre voyelles apparaissent (`e → a`, `a → e`, `e → i`). La dégradation touche toutes les catégories de caractères et 371 lignes sur 574 (122 seulement s'améliorent) : ce n'est pas l'effet de quelques lignes aberrantes. Explication probable : l'agrandissement par interpolation n'ajoute aucune information et étale chaque lettre sur des tokens plus flous, que l'encodeur visuel, gelé, ne peut pas apprendre à interpréter. Le goulot d'étranglement est donc **ce que voit l'encodeur**, pas la résolution.
9. **Adapter l'encodeur visuel confirme ce diagnostic : le CER passe de 22,1 % à 7,9 %**, avec un seul changement par rapport à l'expérience de référence (LoRA ajouté aux blocs de l'encodeur visuel et au projecteur vision-langage ; images d'origine ; mêmes réglages d'entraînement). Les substitutions tombent de 13,5 % à 4,2 % et les confusions vers `i` de 254 à 73. Les erreurs restantes de Qwen ressemblent à celles de Kraken (`l → s`, `n → u`, `c → t`, `n → m`) : elles tiennent désormais surtout aux ambiguïtés de l'écriture.
10. **Qwen adapté et Kraken affiné sont statistiquement à égalité sur l'ensemble du test.** Bootstrap apparié sur les lignes (10 000 rééchantillonnages) : écart de CER de +0,7 point en faveur de Kraken, intervalle à 95 % [−0,4 ; +2,2] ; Qwen fait mieux dans 12 % des rééchantillonnages. Par manuscrit, Kraken est significativement meilleur sur `additional-21218` (+2,6 points, IC [+0,5 ; +6,1]), et Qwen meilleur sur `naf-6195` aux lignes inclinées (−0,9 point, IC [−2,2 ; +0,3], non significatif). Qwen lit mieux les abréviations (57,8 % contre 52,9 %) et les espaces, Kraken un peu mieux les lettres ordinaires (94,4 % contre 93,7 %).

## Données

Corpus **HTRogène – Occitan** (Biblissima+) : 19 manuscrits, 100 pages, 4 293 lignes transcrites (132 lignes vides écartées), annotées en ALTO selon les conventions CATMuS/Segmonto.

> Wiedner, M., Couffignal, G., Chagué, A., Clérice, T. *HTRogène, Occitan corpus of ground-truth for Handwritten Text Recognition and Layout Segmentation* [Data set]. https://github.com/HTRogene/occitan — licence CC-BY 4.0.

**Découpage par manuscrit**, pour mesurer la généralisation à des mains inconnues :

| Split | Manuscrits | Lignes |
|---|---|---|
| Train | 13 manuscrits | 3 392 |
| Dev | additional-10323, paris-bnf-fr-13504, london-bl-harley-3041 | 327 |
| Test | paris-bnf-naf-6195, additional-21218, paris-bnf-arsenal-6355 | 574 |

Le manuscrit `paris-bnf-fr-13509` figure aussi dans le jeu CATMuS Medieval, sur lequel le modèle Kraken de départ a été entraîné : il est placé dans le train pour éviter toute contamination du test.

## Méthode

- **Découpage des lignes** : à partir des polygones ALTO, chaque ligne est extraite et tout ce qui sort de son polygone est masqué en blanc.
- **Évaluation** : CER calculé sur l'ensemble du corpus de test (total des erreurs / total des caractères, avec `jiwer`), après normalisation Unicode **NFD** des deux côtés, comme pour l'entraînement du modèle CATMuS.
- **Kraken** : modèle [CATMuS Medieval 1.5.0](https://doi.org/10.5281/zenodo.12743230) (Kraken 7.1.1). Affinage avec extension de l'alphabet (`--resize union`), taux d'apprentissage 1e-4, lots de 16, augmentation d'images ; meilleur checkpoint choisi sur le dev (époque 9, précision dev 0,886).
- **Qwen2.5-VL-3B-Instruct** : LoRA (r = 16, α = 32) sur les couches d'attention et MLP du **modèle de langue** (0,79 % des paramètres entraînés) puis, dans l'expérience finale, aussi sur les blocs de l'**encodeur visuel** (`attn.qkv`, `attn.proj`, MLP) et sur les deux couches linéaires du **projecteur** vision-langage (1,09 % des paramètres entraînés) ; 2 époques, taux d'apprentissage 2e-4 (cosinus), lots effectifs de 16 lignes, fp16 ; seule la réponse est apprise (consigne et image masquées dans les labels). Génération déterministe (`do_sample=False`).
- **Matériel** : GPU NVIDIA T4 (Google Colab, Kaggle).

## Export XML-TEI

Les transcriptions de Kraken affiné sont exportées en TEI, un fichier par manuscrit de test (`tei/`), **valides contre `tei_all`** :

- `<teiHeader>` : source (corpus HTRogène, licence), modèle et CER mesuré sur le manuscrit ;
- `<facsimile>` : une `<surface>` par page, une `<zone>` par ligne avec son polygone ;
- `<body>` : transcription diplomatique, un `<lb facs="#…"/>` par ligne du manuscrit ;
- zones Segmonto converties lorsqu'elles sont annotées : `NumberingZone` → `fw[@type='folNum']`, `RunningTitleZone` → `fw[@type='header']`, `MarginTextZone` → `note[@place='margin']`, `HeadingLine` → `seg[@type='heading']`.

## Limites

- **Test de petite taille** : 574 lignes et 3 manuscrits ; les CER par manuscrit reposent sur 93 à 337 lignes. Le bootstrap rééchantillonne les lignes, qui ne sont pas indépendantes au sein d'un manuscrit : les intervalles sont probablement trop étroits, et 3 manuscrits ne suffisent pas à généraliser à d'autres mains.
- **Une seule exécution par configuration** : la variabilité due à l'initialisation des adaptateurs LoRA et à l'ordre des données n'est pas mesurée.
- **Entraînement de Kraken arrêté à l'époque 9**, alors que la précision sur le dev progressait encore légèrement.
- **Lignes non redressées** : Kraken a été affiné sur des images de lignes déjà découpées, alors que le modèle de départ avait été entraîné sur des lignes redressées à partir de leur ligne de base.
- **Expérience ×2 pas strictement isolée** : pour tenir en mémoire, les lots par GPU sont passés de 2 à 1 ligne (lot effectif inchangé : 16 lignes), et 393 lignes du train (11,6 %) ont été agrandies moins que ×2 (plafond d'un million de pixels).
- **Annotation des zones hétérogène** selon les manuscrits : le paratexte (ex-libris, titres) n'est pas toujours distingué du texte dans la TEI.

## Pistes

- combiner les deux modèles (vote, ou correction de la sortie de Kraken par Qwen), puisque leurs forces diffèrent selon les manuscrits et les catégories de caractères ;
- mesurer la variabilité en répétant les entraînements avec plusieurs graines ;
- affinage de Kraken directement sur l'ALTO (lignes redressées) ;
- pré-entraînement sur CATMuS Medieval (ancien français, latin) avant l'occitan ;
- approche hybride : transcription Kraken corrigée par un modèle de langue.

## Structure du dépôt

```
notebooks/   préparation des données, Kraken, Qwen, analyse et export TEI
results/     prédictions sur le test (CSV) et analyse d'erreurs (JSON)
tei/         transcriptions TEI des manuscrits de test
```

Les images et les poids des modèles ne sont pas versionnés : les notebooks les régénèrent à partir du dépôt HTRogène et des modèles publics.

## Licence

- **Code** (notebooks) : licence MIT, voir [LICENSE](LICENSE).
- **Transcriptions, prédictions et fichiers TEI** : dérivés du corpus HTRogène, diffusés sous **CC-BY 4.0** ; toute réutilisation doit citer le corpus HTRogène (voir *Données*).

## Auteur

Ezéchiel Sawadogo — Master Langue et Informatique (TAL), Sorbonne Université · [GitHub](https://github.com/ezechiel-sawadogo-nlp) · [LinkedIn](https://www.linkedin.com/in/ezechiel-sawadogo/)
