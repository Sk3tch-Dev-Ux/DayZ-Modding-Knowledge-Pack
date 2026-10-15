# tR_* mods: base-building patterns (extraction, 2026-09-30)

These patterns come from a commercial base-building framework by **Tierro** (Steam `76561198346095154`): tR_Dependence (the framework), tR_Fortification, tR_Stockade, tR_Modules, tR_TreeHouse and tR_ExplosiveCharge, plus the server owner's tR_Custom override pack. The PBOs were recovered with `japm-pbo-recovery`. Identifiers are obfuscated, so method and member names below are described by role. String literals, configs and memory points are original.

Evidence paths are relative to `D:\Cursor Knowledge Dive\`. Finding IDs point into `D:\Cursor Knowledge Dive\_tR_sweep\_state\findings.jsonl`. Verified against vanilla **1.29** scripts (`P:\scripts`). The >93-part hooks were re-checked statically against 1.30 Exp `scripts.pbo` (pass 2) and are unchanged.

Treat this as a design catalogue, not copy-paste source. The code has many bugs (§12), and the mods are someone else's licensed product.

---

## 1. Framework layering: a config flag, not inheritance (A_001, A_002, A_026)

- Structures extend vanilla `BaseBuildingBase` directly. Every override of a *shared* vanilla class (`Construction`, `ActionDismantlePart`, `ActionCloseFence`, `CombinationLock`, `CodeLock`, Expansion and HDSN shims) branches on `ConfigGetBool("tR_BaseBuilding")` and falls through to `super` otherwise. `tR_Dependence/Scripts/4_World/Construction.c:6,55,86,168`
- The modded `BaseBuildingBase` declares a stub virtual API that each structure fills in: `tR_CanOpenGates`, `tR_StartOpenGates`, `tR_StartCloseGates`, `tR_IsLocked`, `tR_HasFullyConstructedGate`, `tR_SnapPointCount(type)`, `tR_GetlvlName`, `tR_IsOverConstructed`, `tR_CreateDamageZones`, `tR_GetDoorComponents`, `tR_GenerateSnow`. `tR_Dependence/Scripts/4_World/BaseBuildingBase.c:137-198,386-406`
- **`Construction.ExceptionCheck` must return false** on a non-Fence structure that has gate parts. Vanilla calls `GetConstructionPart("wall_base_down").IsBuilt()` for every `IsGate()` part and dereferences null. `Construction.c:53-63`
- **Vanilla CombinationLock without a Fence.** `UnlockServer` is reimplemented to drop the lock at the structure position (`ServerDropEntity`, `PlaceOnSurface`, `SetTakeable`). The dial actions accept a lock whose parent is a tR structure. `CombinationLock.c:3-71`
- Scope every such override to the flag. `Construction.IsColliding` / `IsCollidingEx` were left unscoped, so vanilla fences and watchtowers stopped colliding with anything that isn't a `BaseBuildingBase` (A_007). `Construction.c:166-242`

## 2. More than 93 parts (C_001, E_001)

160 parts in one structure. Ids 1..93 go to `super`. Ids 94..124, 125..155 and 156..186 go to three extra `RegisterNetSyncVariableInt` ints (`mask = 1 << (id - 94|125|156)`) through overrides of all three hooks:

```c
override void RegisterPartForSync(int id)    { if (id <= 93) super.RegisterPartForSync(id); else if (id <= 124) m_Sync4 |= 1 << (id - 94); /* ... */ }
override void UnregisterPartForSync(int id)  { /* mirror with &= ~mask */ }
override bool IsPartBuildInSyncData(int id)  { if (id <= 93) return super.IsPartBuildInSyncData(id); /* test the extra ints */ }
```

- It works because vanilla `SetPartFromSyncData` reads `IsPartBuildInSyncData(part.GetId())` (1.29 `basebuildingbase.c:280`). `tR_Fortification/Scripts/4_World/tR_FortificationKit.c:101-112,1973-2054`
- Persistence: after `super.OnStoreSave` the order is gate state, `m_IsOpened`, int4, int5, int6. Reads come back in the same order, and every failed read returns false. There is no tR-owned version marker, only Fence's legacy `version < 110` branch. `tR_FortificationKit.c:1912-1960`. R9-audit it before shipping (`persistence-audit.md`).
- **Stable sparse ids across sibling structures.** StockadeCorner reuses Stockade's ids for every shared part and leaves out the rest (skipping 7, 8, 17-19 and 22), so the scripts stay diff-identical (C_021).

## 3. Part graph grammar (C_002, C_003, C_009, C_028, C_034, D_008, D_028)

```
tr_level_1_foundation (is_base) -> tr_level_1_base
tr_level_N_base | tr_level_N_roof_base        (conflict; both REQ tr_level_(N-1)_base; level 4 = roof only)
tr_level_N_wall_M_base | _doorway_base | _gateway_base         (frame alternatives, conflict)
   -> _wood | _metal | _cut_wood | _cut_metal                  (finish clique, all conflict)
   -> _sashes  (REQ cut, conflicts only the solid finishes)
roof_base unlocks gable parts on walls 1 and 3 only; script hides walls 2 and 4 under a roof
ceiling | ceiling_cut (conflict) -> stairs (REQ ceiling_cut, conflicts wall_3 cut/sashes)
furniture_storage | furniture_workbench (conflict each other and that wall's cut/sashes)
```

`tR_Fortification/config.cpp:3194-5574`; `tR_FortificationKit.c:239-264,1367-1393`

- **Per-floor inventory.** `CanDisplayAttachmentCategory` / `CanDisplayAttachmentSlot` show a floor's categories and slots only when its prerequisite part is built **and** the player is within 0.5-2.0 m vertically of memory point `tr_level_N`. A Watchtower clone scaled to 4 levels × 4 walls. `tR_FortificationKit.c:1340-1571`
- `platform_support = "tr_level_N"` matches the lowercased category prefix, so vanilla `DropNonUsableMaterialsServer` empties a floor's slots when its support part goes (C_034).
- **Built-in furniture as parts.** Storage (9 crate slots) and a workbench (29 tool display slots). On dismantle the hardcoded slot arrays drop to `<part>_materials` (C_028).
- **Spikes** are roof-top parts that conflict with the next storey's base.

## 4. Memory-point snapping (A_012-A_016, A_019-A_021, B_024, C_015, D_021)

**Kit side (placing class config):**
```cpp
class tR_NarrowDoorKitPlacing { tR_CanSnapping=1; tR_RotationAngle=0; tR_PointName="tR_NarrowDoorKitPlacing";
    tR_TargetIsMatch[]={"tR_Fortification","tR_MediumDoor","tR_TreeHouse"}; alignHologramToTerain=0; slopeTolerance=1; };
```
**Structure side:**
- `tR_CanSnapping=1` in config.
- Memory points `point_<pointname lowercased>_<i>`, each with `<snap>_collision_min/_max` (Stockade also uses `_collision_axis` and `_collision_1_min/_max`).
- Script `tR_SnapPointCount(type)` says how many points to scan: Fortification 4, 8 for narrow doors, 12 for the garage door; Triangle 3; Stockade 2. `tR_Fortification/Scripts/4_World/tR_FortificationKit.c:307-321`

**Runtime (client):**
- A 10 m camera ray, `DayZPhysics.RaycastRVProxy(..., CollisionFlags.ALLOBJECTS, projection)`, picks the first `BaseBuildingBase` with `tR_CanSnapping` whose type is in `tR_TargetIsMatch[]`. It then takes the nearest `point_*`.
- Yaw = target yaw plus an offset from a table keyed by (kit type, target type, snap index), falling back to `target yaw + rotationSteps × tR_RotationAngle`. Wrap-safe smoothing: lerp 0.07 per frame, unwrapping by ±360 when the jump exceeds 100°.
- The snap is rejected if `IsBoxCollidingGeometry` on the snap box hits another `BaseBuildingBase`.
- `UAMeleeAttackModifier` + prev/next action nudges height by ±0.03 m (wraps at ±1 m).
- Snapping on/off is a **local, instant deployment action** (`IsLocal()`, `IsInstant()`, `IsDeploymentAction()` all true).

`tR_Dependence/Scripts/4_World/yqxT1UcjCi2HXLp.c:9-332`; `BaseBuildingBase.c:55-97,211-385`; `tR_ActionSnappingSwitch.c:1-73`

**Wall-slide** (`tR_CanSlide`, used by charges): `SphereCastBullet` with radius 0.05 from the eye on `BUILDING|FENCE|ITEM_LARGE`. If `abs(normal.y) < 0.7`, yaw is `Atan2` of the flattened inverse normal and the pose is pulled 0.08 m toward the player (A_020).

**Don't copy:**
- Index matching with `Contains("_1")`, which also matches `_10`-`_12` (A_014).
- Replacing the vanilla `EvaluateCollision` verdict (A_017).
- Running 2-4 box queries every frame from both `SetProjectionOrientation` and `SetProjectionPosition` (A_016).

## 5. Doors, gates and sashes (C_007, C_022, D_001-D_006, D_009, D_010, D_018, C_030)

- **Animation phase as degrees.** `AnimRotate { source=user; animPeriod=0.018; }` driven to 60, 80 or 100 (leaves), 50 (sashes `_translation`) and 5 (lock handle and hasp). Arrival is checked with exact `==`. Verified in pass 2 from the ODOL v55 animation records: Stockade 0..100, MediumDoor 0..80, sashes 0..50, lock 0..5, so `animPeriod` is seconds per unit of phase. The garage door uses `animPeriod=3` with phases 0..1. See `dayz-animation-pipeline`.
- **Gate state from parts, not `is_gate`.** `CheckGateState()` returns 0, 1 or 2 from `IsPartConstructed` of the hinge or leaf bases. It is a synced int, recomputed in `OnPartBuilt/Dismantled/DestroyedServer` and `AfterStoreLoad`. 1 means hinges (can open); max means a full gate (can take a lock). No part sets `is_gate`.
- **Two-stage open.** The server animates handle and hasp to 5 and sets a synced "lock" bool. A `Timer.Run(0.3, this, "OpenFence")` then swings the leaf and sets `m_IsOpened`. Clients replay both stages by diffing each synced value against a local shadow copy in `OnVariablesSynchronized`. Lock proxies and camonets have their own `_Rotate_To_Gate` sources driven to the leaf phase, so they follow the door.
- **Completion** is polled with `CallLater(Check..., 0, true)` every frame, removed once the phase arrives. `UpdateNavmesh` runs via `CallLater` 2000-3000 ms after a swing.
- **Sashes API** works on any structure: `tR_OpenSashes(part)` sets `part + "_translation"` to 50 (closed is 0). Actions find an action component containing `_sashes` and work only from inside (`IsPlayerInside`). They emit AI noise via `NoiseParams.LoadFromPath("CfgVehicles SurvivorBase NoiseActionDefault")`. The shed's cargo opens only when its hatch phase is 50 and the player stands at floor height (`CanDisplayCargo`, D_010).
- **Garage roller door** (D_018, D_019): a monolithic `BaseBuildingBase` with no `Construction{}`. One source moves about 48 slat bones. Only the `tR_Rollets` zone has `transferToGlobalCoef=1`, and at global `RUINED` the door calls `DeleteSafe()`. The lock attaches only while the rollets phase is 0.

Evidence: `tR_Modules/Scripts/4_World/tR_MediumDoorKit.c:98-105,346-373,593-785`; `tR_GarageDoorKit.c:1-351`; `tR_Fortification/Scripts/4_World/tR_FortificationKit.c:906-1032,1288-1335`

## 6. Per-part damage zones and area damage (C_004, C_005, C_031, D_011, D_019, D_020)

- **Per-part `AreaDamageLoopedDeferred_NoVehicle`** for spikes and palisades. Extents come from `<part>_damage_min/_max`. The trigger is **spawned 1000 m below** (`SetAreaPosition(center + "0 -1000 0")` before `Spawn()`) and then moved to `center`, so nobody is hit during setup. Zones live in `map<string, ref AreaDamageManager>` keyed by part, are rebuilt in `AfterStoreLoad` and destroyed on dismantle, destroy and `EEDelete`. `tR_Fortification/Scripts/4_World/tR_FortificationKit.c:1033-1094`
- **The zone follows a swinging gate** by maths rather than attachment: rotate the centre around `ModelToWorld("<part>_axis")` in XZ and subtract the angle from yaw. Zones are deleted while the leaf moves and recreated at rest (`tR_Stockade/Scripts/4_World/tR_StockadeKit.c:675-751`). The pitch variant for roofs is wrong (C_006).
- **DamageSystem tuned for raids.** `GlobalHealth` 100 with zero armour, every part zone `transferToGlobalCoef=0` with 12k-25k HP, zone class name = part name, and inheritance to compress repeats (`class tR_Level_2_Spike: tR_Level_1_Spike { componentNames[]=...; }`).

## 7. Locks and third-party compatibility (A_027, A_030, A_031, A_033, C_023, D_013, B_027, E_002)

- **One slot, three lock types.** `Att_CombinationLock` accepts `CombinationLock`, `CodeLock` (`#ifdef CodeLock`, plus `GetCodeLockConfig().CanAttachToGates()`) and `ExpansionCodeLock` (`#ifdef DZ_Expansion_BaseBuilding`). The visual is picked with `SetSimpleHiddenSelectionState(0|1|2)` over `att_combinationlock|att_codelock|att_expansion_codelock`.
- CodeLock actions are filtered by action-component selection `gate_interact`. `CodeLockServerRPC.OpenTarget` is redirected to `tR_StartOpenGates`.
- **Expansion:** `AddAction`/`RemoveAction` overrides chain the Expansion code-lock actions onto the custom open action. `CF_OnStoreLoad` reads `storage[DZ_Expansion_BaseBuilding]` and replays Expansion's legacy (<38) layout.
- **Breaching mods:**
  - HDSN `BreachingCharge`: `GetWallType` returns a synthetic type string derived from built parts (`_HalfMetal`, `_FullMetal`); `DestroyTarget` destroys `tR_GetDoorComponents()`.
  - KB C4: planted at memory point `tr_explosivecharge_pos`.
  - Both shims have bugs (§12).
- `requiredAddons` lists only vanilla, so load order relative to these mods depends on the launch line (A_042).

## 8. Kits, dismantle, repair, variants (B_019, B_020, B_023, C_011, C_012, D_016, D_017, D_022)

- **Deploy:** a WatchtowerKit clone (`CreateObjectEx(..., ECE_PLACE_ON_SURFACE)`, then set the hologram pose, `HideAllSelections`, `UpdateVisuals`). Generic `tR_KitBase` alternative: building class = kit type with `"Kit"` removed, and the kit's health level is copied to every zone.
- **Dismantle the whole structure back to a kit:** `tR_ActionDismantle` (10 s) checks that the hand item's type is in config `tR_DismantleTools[]` and that the structure is unlocked and closed. It creates `GetType()+"Kit"`, copies the health level, wears the tool by one level and calls `DeleteSafe()` on the structure.
- **Blowtorch repair:** `CAContinuousRepeat(20 s)`, `CMD_ACTIONFB_FLAME_REPAIR`. It switches the blowtorch on at `OnStartAnimationLoopServer` and steps health up one level per cycle (but flattens every zone, see B_021).
- **Kit variant toggle in hand:** a 1 s crouch action using `ServerReplaceItemInHandsWithNew(new ReplaceItemWithNewLambdaBase(old, siblingType))`. One recipe gives both Stockade and StockadeCorner, or Narrow and Medium door.
- **Winter edition:** server JSON `EnableWinterEdition` picks `tR_XWinter` (an empty script subclass with only a different model) at deploy. Snow overlays are separate non-persistent `Building`s spawned in `AfterStoreLoad`, kept in a map keyed `<part>_snow`, and deleted on dismantle, on the next storey and in `EEDelete`.

## 9. Structures anchored to world objects (D_024-D_027, D_031)

- **Tree house:**
  - A pickaxe action, valid only on `TreeHard_t_PiceaAbies_3f(_summer)` with no `BaseBuildingBase` within 5 m, creates `tR_TreeHouse` at `ModelToWorld(tree.GetMemoryPointPos("action")) - 1.5 m` and turns it to face the player. There is no kit.
  - Modded `ActionMineTree` refuses to fell a tree with any `BaseBuildingBase` within 2 m of its `action` point.
  - Removal is a separate pickaxe action gated by `CanFoldBaseBuildingObject()`.
  - `tR_Dependence/Scripts/4_World/tR_ActionDigAreaCB.c:1-73`; `tR_TreeHouse/Scripts/4_World/ActionMineTree.c:1-26`
- **Climbable ladder:** a `BaseBuildingBase` can't host engine ladder points, so building part `tr_ladder` spawns a companion `tR_Ladder_Building` (`Building`, scope 1) whose model carries `ladder1_bottom_front`, `ladder1_top_left`, `ladder1_dir`, `ladder1_con` and `ladder1_con_dir`. It is deleted on dismantle and in `EEDelete`, but **not** when the part is destroyed in a raid, and it is created in `UpdateVisuals` without a server check (D_031, unverified).

## 10. Attachments and visuals (A_036, A_037, A_039, B_031, C_017, C_026, C_027, C_029)

- **Vanilla items become attachable** through `inventorySlot[] +=` on WoodenLog, WoodenPlank, Nail, MetalPlate, CamoNet, crates, tools, grenades, AmmoBox and `Rifle_Base` (weapon rack), and `ActionAttachToConstruction` is added to their `SetActions`. There are 173 `ProxytR_<slot>` `CfgNonAIVehicles` classes whose proxy file names equal the slot names.
- **Memory-point contract consumed by script:**
  - `<part>_materials` is the ground drop point (`InventoryLocation.SetGround` + `TakeToDst(InventoryMode.SERVER)` at `ModelToWorld`);
  - `<part>_min/_max` is collision;
  - `<part>_damage_min/_max` is area damage;
  - `<part>_axis` is the pivot;
  - `tr_kit_spawn_pos`;
  - `tr_level_0..4` are floor heights;
  - `interact_min/_max`.
- **Slot-name string transforms drive visuals.** `tR_Level_2_Wall_1_Camonet` turns into part prefix and animation-source prefix lookups. This is fragile, but it removes per-slot code.
- **Height-adaptive foundation skirt:** when the structure is more than 3 m above ground, hide `tR_Level_1_Foundation_Fill` and `RemoveProxyPhysics`.
- **Electric items in slots:** the modded `ActionTurnOn/OffWhileOnGround` accept `InventoryLocationType.ATTACHMENT`, so gas lamps work in place. This is a global behaviour change.

## 11. Server-owner override pack (tR_Custom, B_039-B_047)

- One PBO holds one sub-config per target. Each has its own `CfgPatches` whose `requiredAddons` names the target addon, so it loads after it, and it re-declares the classes with the same parent while changing only values.
- The strategy it reveals: zero the Projectile and Melee armour coefficients, then retune HP against the charge damage table, so **only scripted charges raid** (see `dayz-raiding-explosives`).
- Material quantities above slot `stackMax` make parts unbuildable. That's either a hidden disable or a typo, so prefer removing the part.
- Penetration `.bisurf` files live in this pack and the original mods point their `surfaceInfo` at it, which makes the pack mandatory.

## 12. Bug and anti-pattern catalogue (learn from these)

| ID | What | Where |
|---|---|---|
| A_005 | `lock_slot` mutated inside the materials loop, so later materials stay locked on dismantle | `Construction.c:13-49` |
| A_007 | `IsColliding` override not scoped to tR; vanilla fences ignore tents and vehicles | `Construction.c:166-237` |
| A_009/A_010/B_012 | Explosion revert calls super with the new level (ruin still happens); flag never reset, so later melee hits are reverted | `BaseBuildingBase.c:11-53` |
| A_017 | Custom hologram overwrites the vanilla collision verdict | `yqxT1UcjCi2HXLp.c:399-457` |
| A_025 | "Vertical" check uses 3D distance and fails open when the memory point is missing | `BaseBuildingBase.c:149-161` |
| A_032/A_034 | HDSN shim skips super for non-items; KB C4 shim runs super inside a loop over an empty list | `HDSN_DestructionManager.c:4-17`, `KB_ActionPlantC4.c:35-76` |
| A_035 | Bodiless `tR_GetDoorComponents` returns null and `.Count()` is called on it unchecked | `BaseBuildingBase.c:406` |
| B_016 | Hint GUI spawns `ECE_LOCAL` preview entities with physics and never deletes them | `tR_HintGui.c:22-32,155-162` |
| B_021 | Blowtorch repair sets every zone to global level + 1, reviving ruined zones | `tR_ActionRepairBlowTorchCB.c:52-63` |
| C_006 | Roof damage-zone rotation moves the centre in XZ for a pitch | `tR_StockadeKit.c:682-732` |
| C_013/C_014 | Winter triangle class doesn't exist; triangle hologram inherits the wrong kit and has no recipe | `tR_FortificationTriangleKit.c:17-24` |
| C_019/C_020 | `material_type=2` on 18 metal parts (wood SFX); nail counts and conflict lists inconsistent | `tR_Fortification/config.cpp` |
| D_031 | Ladder companion not deleted when the part is raided; created without a server check | `tR_TreeHouse.c:89-96,302-369` |
| D_033 | `IsPlayerInside` discards the `CheckLevelVerticalDistance` result (copied from vanilla) | `tR_MediumDoorKit.c:868-873` |
| D_034 | Client replays `OpenFence` twice, so the gate sound plays twice | `tR_GarageDoorKit.c:167-338` |
| C_008/D_002 | Per-frame `CallLater` pollers with exact float `==` on the animation phase | `tR_FortificationKit.c:1236-1287` |

Cross-references: `dayz-raiding-explosives` (charges, raid rules, client-trust bug), `dayz-doors`, `dayz-animation-pipeline` (phase-as-degrees, AnimationSources templates), `dayz-persistence` (persistent-ID links, appended stream), `dayz-texture-pipeline` (shared `.bisurf`), `enforce-script-reference` (CF RPC config sync, plugin matrix).

---

## 13. Pass 2 additions (2026-09-30): attachment display, dead content, verifications

**Three ways to show attachments in one framework** (P2C_001, P2C_002):
1. **Per-slot stub proxies.** All 169 `ProxytR_<slot>` models are the *same* 464-byte ODOL with no geometry (one md5). A `ProxyAttachment` draws the attached item's own model at the proxy frame, so the stub exists only for its file name. That name becomes the proxy class, and the class lists exactly one slot. Result: a clean 1:1 proxy-to-slot binding at zero mesh cost. `tR_Dependence/config.cpp:1981-3525`
2. **Appending slots to vanilla proxy classes.** `Proxysingle_plank` and `ProxyNails_loose_10` get `inventorySlot[] +=`, and the structure models place those vanilla proxies alongside the stubs. `tR_Dependence/config.cpp:2461-2481`
3. **No proxy class at all.** 30 slots (logs, camonets, wire, canvas) have only a CfgSlots `selection=`. How the logs render is not verified.

**Hand-attach gap** (P2C_006): vanilla `ActionAttach` refuses targets with `CanUseConstruction()` (`ActionAttach.c:66`). Every item you slot onto a structure therefore also needs `ActionAttachToConstruction` in its `SetActions`. tR forgot this for LeatherSewingKit, `Grenade_Base` and `Rifle_Base`, so those can only be dragged in the inventory screen.

**Dead or planned content** (P2C_003, P2C_004):
- `tR_FirstAidKit_0..3` and `tR_SewingKit_3` are fully wired on the item side but used by no structure.
- Eight `tR_Dugout_*` slots, item patches and `#ifdef tR_Dugout` hooks exist for a mod that isn't shipped. It is the same `tR_Dugout` the dirt surface points at.

**Verified in pass 2:**
- **Phase scale:** see §5. The ODOL v55 records give 0..100 / 0..80 / 0..50 / 0..5.
- **The >93-part hooks are unchanged in 1.30 Exp** (`scripts.pbo` `BaseBuildingBase.c:137-269`).
- **`UpdateVisuals()` runs on clients** via `OnSynchronizedClient` (1.29 `:135-144`, 1.30 `:124-133`), so the tree house's unguarded `CreateObject` in `UpdateVisuals` does execute client-side. Whether that yields a local duplicate ladder is the open in-game probe (`P:\TRProbe`).
<!-- tr-mods-extraction pass 2: P2C_001-P2C_007, C_001/C_007/D_003/D_031 verification | 2026-09-30 -->
