# HunterShotTimer - correctif local

Date : 2026-11-06
Version de l'addon : 0.11.3

## Erreur

```
[ADDON_ACTION_BLOCKED] L'AddOn 'HunterShotTimer' a tente d'appeler la fonction protegee 'Button:SetPassThroughButtons()'.
[C]: in function 'SetPassThroughButtons'
Blizzard_MapCanvas/MapCanvas_DataProviderBase.lua:290: in function 'CheckMouseButtonPassthrough'
Blizzard_MapCanvas/Blizzard_MapCanvas.lua:331: in function 'AcquirePin'
```

## Cause

Blizzard appelle `Button:SetPassThroughButtons()` sur chaque pin de carte, y compris les pins
crees par un addon. Sur un pin cree par un addon, cet appel est protege et bloque :
`ADDON_ACTION_BLOCKED`. Le probleme vient du pin `HunterShotTimerVendorPinTemplate`
(mixin `MapCanvasPinMixin`), pas du reste de l'addon.

## Correctif

Dans `Vendors.lua`, sur `HunterShotTimerVendorPinMixin` :

```lua
function HunterShotTimerVendorPinMixin:SetPassThroughButtons()
end
```

C'est le meme contournement que celui utilise par HereBeDragons pour les pins crees par
des addons. Effet de bord : le clic droit sur le pin ne traverse plus jusqu'a la carte
(le zoom arriere au clic droit peut ne pas fonctionner pile sur le pin). Les clics du pin
et le reste de la carte ne changent pas.

## Verification

- `luaparse` sur les 9 fichiers `.lua` : OK.
- Copie originale : `Vendors.lua.orig` dans ce dossier.
- Deploye dans `Interface\AddOns\HunterShotTimer\Vendors.lua`.

## Si l'addon est mis a jour

Le fichier `Vendors.lua` sera remplace par la nouvelle version. Re-appliquer le no-op
ci-dessus si l'erreur revient.

## Timing model - base sur le vrai swing (2026-11-06)

- Timer.lua : Phase utilise maintenant la vitesse reelle du swing (cycle = speed), sans
  l'extension EarlyDelay.
- ShotAt : modele max(fin du swing, arret + StandNeed()), comme EllesmereUI / Shot Window.
  Arrete tot = tir a la fin du swing ; arrete dans la fenetre ou apres = tir un temps de
  stand-still apres l'arret.
- learn : le temps de stand-still est mesure sur tout arret tardif (dans la fenetre ou
  apres le swing), plus seulement apres le swing.
- Bar.lua : la barre et la vitesse affichee utilisent le swing reel.
- README mis a jour.

Tests : luaparse sur les 9 fichiers = 0 ; test Fengari du modele (arret tot / dans la
fenetre / apres, phases, countdown, stand bar) = OK.

## 2e cause du SetPassThroughButtons bloque - boucle securisee de la carte (2026-11-07)

Meme message d'erreur que ci-dessus, mais pile differente :

```
[C]: in function 'SetPassThroughButtons'
[Blizzard_MapCanvas/MapCanvas_DataProviderBase.lua]:288/290: in function 'CheckMouseButtonPassthrough'
[Blizzard_MapCanvas/Blizzard_MapCanvas.lua]:331: in function 'AcquirePin'
[Blizzard_SharedMapDataProviders/QuestDataProvider.lua]:194: in function 'AddQuest'
[Blizzard_MapCanvas/MapCanvas_DataProviderBase.lua]:76: in function 'OnMapChanged'
[Blizzard_MapCanvas/Blizzard_MapCanvas.lua]:800: in function <Blizzard_MapCanvas.lua:799>
[C]: in function 'secureexecuterange'
```

Cause : la carte appelle le `RefreshAllData` de chaque data provider depuis
`secureexecuterange` (`MapCanvas_DataProviderBase.lua:76` -> `Blizzard_MapCanvas.lua:765`).
Notre provider y appelait `map:AcquirePin(...)`, donc l'execution taint les providers qui
tournent apres le notre (les quetes) et leur appel protege `Button:SetPassThroughButtons()`
est bloque. Le no-op du mixin ne couvre que nos propres pins.

Correctif dans `Vendors.lua`, `provider:RefreshAllData` :

- la construction des pins est deplacee dans `buildPins()` ;
- `RefreshAllData` ne pose plus qu'un `C_Timer.After(0, ...)` : les pins sont crees a la
  frame suivante, hors de la boucle securisee (meme approche que Carrot) ;
- `refreshPending` fusionne les refreshs en rafale (`OnMapChanged` + `RefreshAll`, ou un
  changement d'equipement).

Verification :

- `luaparse` sur `Vendors.lua` : exit 0.
- Test Fengari `lua-check/test-vendorpins.js` qui charge le vrai `Vendors.lua` et simule la
  carte : passe apres le correctif, et echoue sur l'ancien code
  (`no pin may be acquired inside RefreshAllData, got 1`).
- Teste : `refreshPending` -> un seul rebuild pour 3 appels dans la meme frame.

Si l'addon est mis a jour : `Vendors.lua` est remplace, re-appliquer les deux correctifs
(no-op du mixin + defer de `RefreshAllData`).

## 3e cause : ecritures dans les tables de la carte (2026-11-07)

Le defer de `RefreshAllData` ne suffisait pas : l'erreur revient avec la pile
`RefreshAllDataProviders` -> `QuestDataProvider.RefreshAllData` -> `AddQuest` ->
`AcquirePin` -> `CheckMouseButtonPassthrough` -> `SetPassThroughButtons`.

Cause : `AttachVendorMap` ecrivait dans deux tables internes du `WorldMapFrame`, que tous
les pins lisent :

- `WorldMapFrame:SetPinTemplateType(TEMPLATE, "BUTTON")` ecrit dans `pinTemplateTypes` ;
  `AcquirePin` lit cette table pour creer le pool de chaque type de pin. Une valeur tachee
  la donne un `CreateFramePool` tache, donc un pin tache, et l'appel protege
  `Button:SetPassThroughButtons()` est bloque sur les pins de quete.
- `GetPinFrameLevelsManager():InsertFrameLevelAbove("PIN_FRAME_LEVEL_HST_VENDOR", ...)`
  ecrit dans `pinFrameLevelsManager.definitions`, lu par `ApplyFrameLevel` de chaque pin.

Ni HereBeDragons ni HandyNotes ne touchent a ces deux tables : ils utilisent
`UseFrameLevelType("PIN_FRAME_LEVEL_AREA_POI")` et laissent Blizzard deduire le type depuis
le XML (`<Button>`).

Correctif :

- suppression de l'appel `SetPinTemplateType` ;
- suppression du bloc `GetPinFrameLevelsManager` / `InsertFrameLevelAbove` / `AddFrameLevel` ;
- `OnLoad` utilise `"PIN_FRAME_LEVEL_AREA_POI"` au lieu du niveau maison.

Le no-op du mixin et le defer de `RefreshAllData` restent en place.

Verification : `luaparse` exit 0 ; test Fengari `lua-check/test-vendorpins.js` (le stub fait
echouer tout appel a `SetPinTemplateType` ou `GetPinFrameLevelsManager`) : OK sur le nouveau
fichier, echoue sur l'ancien (`must not touch the frame level manager`).

Si l'erreur persiste, etape suivante : no-op `QuestPinMixin.SetPassThroughButtons` comme le
fait ElvUI_WindTools (cout : le clic droit ne traverse plus la carte au-dessus d'un pin de
quete).

## 4e correctif : marchands de munitions aux positions Forever (2026-11-18)

Probleme : `VendorData.lua` venait de bases Classic (pfQuest, Wowhead Classic). Forever a
deplace certaines zones : a Hurlevent, les pins etaient jusqu'a 22 points a cote du marchand
(Thulman Flintcrag), et de 5 a 14 points dans les Carmines, Mulgore et les Maleterres de l'Est.
Il manquait aussi les marchands propres a Forever.

Correctif : `VendorData.lua` est regenere (meme format, `Vendors.lua` inchange) depuis les
faits de Polestar_Codex (spawns des PNJ, roles `sells:arrows` / `sells:bullets`, marchands
de chaque objet), fusionnes avec la liste d'origine de HST :

- positions : celles de Polestar (cadre live de Forever) ; 11 marchands deplaces ;
- 16 marchands ajoutes (dont Nordun Steadysight a Comte-de-l'Or, et ceux de Mount Hyjal,
  Zephras Isle, Riverglades) ;
- objets : ceux de HST, plus ceux que Polestar a vus vendus sur Forever (sources `l`/`e`) ;
  les ajouts venant seulement de Questie/pfQuest sont ignores (0 changement) ;
- aucun marchand de la liste d'origine n'est perdu (139 lignes contre 124).

Generateur : `%TEMP%\opencode\hst-vendors\build-vendordata.js`.
Test : `%TEMP%\opencode\lua-check\test-hst-vendordata.js` charge le vrai `Vendors.lua`
avec les donnees ; il passe sur le nouveau fichier et echoue sur l'ancien.
Original : `VendorData.lua.orig` dans ce dossier.

Si l'addon est mis a jour : relancer le generateur (il relit le `VendorData.lua` de HST et
Polestar_Codex) puis recopier le fichier produit.
