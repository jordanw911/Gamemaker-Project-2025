# Medieval Rancher — GameMaker Project

**Overview**

A top-down GameMaker project that blends elements of Pokémon (raising creatures), Stardew Valley (farming, NPC economy, quests), and Minecraft (resource gathering, building). Players have a medieval plot of land, collect resources (wood, stone, ores, plants), craft tools, build and decorate a home, raise creatures on a ranch, complete quests, explore dungeons, and fight in combat areas.

This document contains: project plan, folder/resource structure, object/room design, data structures for inventory and save system, starter GML scripts (movement, inventory, crafting, mining/tool usage), GUI/drag-and-drop inventory and hotbar, save/load approach, and next-step suggestions.

---

## Milestones

1. Core player movement & animations (top-down) — basic walk, idle, tool-use animations.
2. Tilemap world & collision — walkable areas, obstacles, resources placed as objects or tiles.
3. Mining/harvesting tools & basic resource nodes (trees, rocks) and drop logic.
4. Inventory system (drag & drop) + hotbar + item definitions.
5. Crafting system (recipes) and tool creation.
6. Ranch/creature system (basic creature object, feeding, growth, produce).
7. NPCs, shop, and quest system (buy/sell, reward flow).
8. Save/load system (player state, inventory, world changes, creatures, buildings).
9. Dungeons & combat mechanics (enemies, health, attacks, loot).
10. Home building/decoration and persistence.

---

## Project folder / resource suggestions (GameMaker asset names)

* `Sprites/`

  * `spr_player_idle`, `spr_player_walk`, `spr_player_tool_axe`, `spr_player_tool_pick`, etc.
  * `spr_tree`, `spr_rock`, `spr_ore_*`
  * `spr_item_*` (icons for inventory) — small 32x32 or 48x48
  * `spr_ui_slot`, `spr_ui_hotbar`, `spr_ui_itemstack` (optional)
* `Objects/`

  * `obj_player`
  * `obj_resource_tree`, `obj_resource_rock`, `obj_ore_node`
  * `obj_item` (inventory item logical object)
  * `obj_npc_shop`, `obj_npc_quest`
  * `obj_creature` (raised creatures)
  * `obj_enemy` (dungeon monsters)
* `Rooms/`

  * `rm_main`, `rm_town`, `rm_dungeon_*`, `rm_ranch`
* `Scripts/` (GML functions)

  * `scr_move_player`, `scr_add_item`, `scr_remove_item`, `scr_save_game`, `scr_load_game`, `scr_craft_item`, `scr_use_tool`, `scr_open_inventory`.
* `Tilesets/` for ground types and buildings
* `Fonts/`, `Sounds/`, `Music/`

---

## High-level architecture

* **Data-driven design**: define item and recipe data as ds_maps or JSON files to make it easy to add/remove items.
* **Inventory model**: store inventory as a list of item stacks. Each stack is a ds_map: `{id, qty, metadata}`. Use `ds_list` for ordered inventory slots and `ds_grid` or list-of-maps for equipped/hotbar.
* **GUI**: Inventory and hotbar are drawn in `obj_player` Draw GUI event or a dedicated UI controller `obj_ui_controller`.
* **Save system**: serialize important game state (player position, inventory, recipes unlocked, world node states, creatures) into JSON using `json_encode` and `file_text` or `buffer` functions.
* **Resource nodes**: objects with `hp` that when reduced drop items; they can optionally respawn or be permanently cleared.

---

## Item & Recipe data model (example)

```gml
// scr_init_items()  -- call once at game start (Create event of obj_game_controller)
items = ds_map_create();

// Example: wood
itm = {
    id: "wood",
    name: "Wood",
    stack: 99,
    icon: "spr_item_wood",
    type: "resource"
};
items["wood"] = itm;

// Example: pickaxe
itm = {
    id: "pickaxe_basic",
    name: "Wooden Pickaxe",
    stack: 1,
    icon: "spr_item_pick",
    type: "tool",
    durability: 50
};
items["pickaxe_basic"] = itm;

// recipes (ds_map of recipes keyed by result id)
recipes = ds_map_create();
recipes["pickaxe_basic"] = [
    {id: "wood", qty: 5},
    {id: "rock", qty: 2}
];
```

(Implementation note: GameMaker doesn't allow literal maps/arrays like that directly — use `ds_map_create()` and `ds_list_create()` to populate. The above is pseudocode to illustrate.)

---

## Inventory system (design)

* `player_inventory` is a `ds_list` where each element is a `ds_map` with keys: `id`, `qty`, `meta` (map).
* Inventory size is configurable (e.g., 36 slots) and slots can be empty (store `noone` or map with `id = ""`).
* Hotbar uses first N slots (e.g., 8) or a separate `ds_list` bound to quick slots.
* Drag-and-drop flow:

  * On GUI MouseDown inside inventory slot -> pick up the stack into `held_item` (a map) and set that slot to empty.
  * On MouseUp over a slot -> if `held_item` not empty, attempt to merge or swap with slot.
  * Visuals: draw the item icon under the mouse while `held_item`.

### Starter GML for inventory structure

```gml
// Create event (obj_player)
inventory_size = 36;
player_inventory = ds_list_create();
for (var i = 0; i < inventory_size; ++i) {
    ds_list_add(player_inventory, ""); // empty slot
}

hotbar_size = 8;
// hotbar just maps to first 8 slots by default
hotbar_index = 0; // selected slot index 0..hotbar_size-1
held_item = ""; // the stack being dragged
```

### Add item function (script)

```gml
/// scr_add_item(item_id, amount)
function scr_add_item(_id, _qty) {
    // try to stack into existing slots
    for (var i = 0; i < ds_list_size(player_inventory); ++i) {
        var slot = player_inventory[| i];
        if (slot != "") {
            if (slot.id == _id && slot.qty < items[_id].stack) {
                var can_add = min(_qty, items[_id].stack - slot.qty);
                slot.qty += can_add;
                player_inventory[| i] = slot;
                _qty -= can_add;
                if (_qty <= 0) return true;
            }
        }
    }
    // place into empty slots
    for (var i = 0; i < ds_list_size(player_inventory); ++i) {
        var slot = player_inventory[| i];
        if (slot == "") {
            var newStack = ds_map_create();
            newStack[? "id"] = _id;
            newStack[? "qty"] = min(_qty, items[_id].stack);
            player_inventory[| i] = newStack;
            _qty -= newStack[? "qty"];
            if (_qty <= 0) return true;
        }
    }
    // remaining amount couldn't be added
    return false;
}
```

(Notes: Use ds_map functions rather than direct `slot.id` access depending on GameMaker version.)

---

## Drag & Drop GUI basics (Draw GUI + Mouse events)

* Use `obj_ui_controller` with Draw GUI event to render slots and icons and to manage mouse interactions.
* Keep `gui_mouse_x`, `gui_mouse_y` in mind (GUI coordinates) and scale with display if needed.
* Example events to implement: `Mouse Left Pressed`, `Mouse Left Released` inside the UI controller.

**Draw GUI pseudocode**

```gml
// Draw GUI
for slot_index in 0..inventory_size-1 {
    var x = slot_x + (slot_index mod cols) * slot_w;
    var y = slot_y + floor(slot_index / cols) * slot_h;
    draw_sprite(spr_ui_slot, 0, x, y);
    var stack = player_inventory[| slot_index];
    if (stack != "") {
        draw_sprite_ext(asset_get_index(stack[? "icon"]), 0, x, y, 1,1,0, c_white, 1);
        draw_text(x + 18, y + 18, string(stack[? "qty"]));
    }
}

if (held_item != "") {
    draw_sprite_ext(asset_get_index(held_item[? "icon"]), 0, gui_mouse_x, gui_mouse_y, 1,1,0, c_white, 1);
}
```

---

## Hotbar & Item Use

* Hotbar displays N quick slots. Player can change selected slot via mouse wheel or number keys.
* `use_item()` script handles using items or equipping tools. Tools change `player.current_tool` and trigger different interactions when using (mining, chopping, digging).

```gml
/// scr_use_hotbar(slot_index)
var slot = player_inventory[| slot_index];
if (slot != "") {
    var id = slot[? "id"];
    if (items[id].type == "tool") {
        player.current_tool = id;
    } else if (items[id].type == "consumable") {
        // apply effect, decrease qty
        slot[? "qty"] -= 1;
        if (slot[? "qty"] <= 0) {
            ds_map_destroy(slot);
            player_inventory[| slot_index] = "";
        } else {
            player_inventory[| slot_index] = slot;
        }
    }
}
```

---

## Player Movement & Animation (top-down)

* Use keyboard WASD or arrow keys. For 8-directional movement, set sprite angle/animation states.
* Movement script example (obj_player Step event):

```gml
// scr_move_player()
var h = keyboard_check(ord('D')) - keyboard_check(ord('A'));
var v = keyboard_check(ord('S')) - keyboard_check(ord('W'));
var speed = 3;
if (h != 0 || v != 0) {
    var len = point_distance(0,0,h,v);
    h /= len; v /= len; // normalize
    x += h * speed;
    y += v * speed;
    // set sprite/animation based on direction
    if (abs(h) > abs(v)) {
        if (h > 0) sprite_index = spr_player_walk_right;
        else sprite_index = spr_player_walk_left;
    } else {
        if (v > 0) sprite_index = spr_player_walk_down;
        else sprite_index = spr_player_walk_up;
    }
} else {
    // idle
    // set idle sprite based on last dir
}
```

* For smoother tile/grid movement, optionally snap to grid when not moving.

---

## Tool use & Resource nodes

* Each resource node (e.g., `obj_resource_tree`) has `hp` and `required_tool` (e.g., `axe`). When player uses the correct tool within range and triggers `use_tool()` (mouse click), reduce `hp` by a value depending on tool tier and possibly player stats.
* On `hp <= 0`, spawn resource item(s) into the world or call `scr_add_item("wood", amount)` and optionally create a visual drop.

```gml
// obj_resource_tree - Create
ohps = 10;
required_tool = "axe";

// When player uses an axe and hits
if (other_tool == required_tool) {
    hp -= tool_damage;
    if (hp <= 0) {
        scr_add_item("wood", 3);
        instance_destroy();
    }
}
```

---

## Crafting system

* Crafting UI reads `recipes` and checks `scr_has_resources(recipe_list)` to enable craft button. On craft, call `scr_remove_item(id, qty)` for each ingredient and `scr_add_item(result_id, qty)` to give the crafted item.

---

## Save / Load (persistence)

* Save player position, inventory contents, player stats, world node states, creatures and their states, built structures, and quest progress.
* Example approach (JSON):

  * Create a `save_data` ds_map containing keys: `player` (map: position, money, inventory as list-of-maps), `world_nodes` (list of nodes with id and state), `creatures` (list), `buildings`, `quest_states`.
  * Use `json_encode(save_data)` and `file_text_open_write` / `file_text_write_string` / `file_text_close`.

**Simple save example**

```gml
/// scr_save_game(filename)
var save = ds_map_create();
// Player
var p = ds_map_create();
p["x"] = x; p["y"] = y; p["money"] = money;
// inventory: convert to a data array
var inv_list = ds_list_create();
for (var i=0; i<ds_list_size(player_inventory); ++i) {
    var slot = player_inventory[| i];
    if (slot == "") ds_list_add(inv_list, "");
    else { var m = ds_map_create(); m["id"] = slot[? "id"]; m["qty"] = slot[? "qty"]; ds_list_add(inv_list, m); }
}
p["inventory"] = inv_list;
save["player"] = p;

var json = json_encode(save);
var file = file_text_open_write("savegame.sav");
file_text_write_string(file, json);
file_text_close(file);
```

(When loading, reverse the process using `json_decode` or parse JSON into ds_maps/lists.)

---

## Creature / Ranch Basics

* `obj_creature` has states: baby, juvenile, adult; needs: food, shelter. Growth occurs over time (timers). Produce yield when adult (e.g., milk, wool).
* Players can place creature objects on ranch tiles. Each creature stores metadata (type, hunger, affection, growth_progress).

---

## NPCs, Shop & Quest System

* NPCs are objects with a dialogue script. For shops, give an `inventory` list (map of item ids and prices). Interaction opens shop UI letting player buy/sell.
* Quests stored as `quest_id` keyed maps with `progress`, `is_complete`, `requirements`. Completion gives experience, money, or recipe unlocks.

---

## Dungeons & Combat

* Create dungeon rooms with enemy spawn logic. Use `obj_enemy` with behaviors (patrol, chase, attack). Implement player HP, damage, invulnerability frames and loot drops.
* For procedural dungeons, use room templates or simple algorithm to place rooms and corridors (advanced).

---

## Tips for using custom sprites & animations

* Keep consistent pivot points (origin) for top-down sprites. Typically center origin or bottom-center for characters.
* Use separate sprites/animations for different tool-use directions (player swing up/down/left/right).
* Export icons for UI at consistent sizes.

---

## Performance & Data notes

* Use `ds_clear` and `ds_destroy` appropriately to avoid memory leaks when replacing or destroying maps/lists.
* When saving many objects, only persist data necessary to reconstruct them (type, position, state) rather than full runtime objects.

---

## Starter checklist — which to implement first?

* [ ] Create player object with movement/animations
* [ ] Create `obj_ui_controller` and simple inventory draw
* [ ] Implement `scr_add_item` & `scr_remove_item` and test adding items
* [ ] Implement a tree object (resource node) and tool use
* [ ] Implement save/load simple player state

---

## Example: Minimal working Player movement + simple inventory add (GML code)

```gml
// obj_player - Create
move_speed = 3;
player_inventory = ds_list_create();
for (var i=0; i<36; ++i) ds_list_add(player_inventory, "");

// Step event (movement)
var hx = keyboard_check(ord('D')) - keyboard_check(ord('A'));
var vy = keyboard_check(ord('S')) - keyboard_check(ord('W'));
if (hx != 0 || vy != 0) {
    var len = point_distance(0,0, hx, vy);
    x += hx/len * move_speed;
    y += vy/len * move_speed;
}

// Example adding wood for debugging
if (keyboard_check_pressed(ord('G'))) {
    scr_add_item("wood", 5);
}

// scr_add_item script (simplified)
function scr_add_item(_id, _qty) {
    // stack or place
    for (var i=0; i<ds_list_size(player_inventory); ++i) {
        var slot = player_inventory[| i];
        if (slot != "" && slot[? "id"] == _id) {
            slot[? "qty"] += _qty; player_inventory[| i] = slot; return true;
        }
    }
    // find empty slot
    for (var i=0; i<ds_list_size(player_inventory); ++i) {
        var slot = player_inventory[| i];
        if (slot == "") {
            var m = ds_map_create(); m["id"] = _id; m["qty"] = _qty; player_inventory[| i] = m; return true;
        }
    }
    return false;
}
```

---

## Next steps & how I can help now

I can:

* Provide fully-detailed GML implementations for any subsystem (inventory GUI drag-drop; save/load; crafting UI; tool-use & mining; creature lifecycle; NPC shop & quest logic; dungeon generation; combat mechanics).
* Create step-by-step tasks with code you can paste into GameMaker.
* Design data files (JSON / included scripts) to hold items & recipes.

Tell me which subsystem you want to start implementing first (e.g., **Inventory GUI + Drag & Drop**, **Player Movement & Animations**, **Save/Load System**, **Crafting & Tools**, or **Creature/Ranch system**) and I will generate the full GML code and explain where to place it in your GameMaker project.

---

*End of document.*
