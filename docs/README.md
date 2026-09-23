# Photon Realtime GDExtension — Documentation

Welcome to the documentation for the **Photon Realtime** plugin for Godot 4.3+. This plugin integrates the Photon C++ SDK (LoadBalancing API) via GDExtension technology, providing maximum performance for object synchronization, RPCs, and room management.

> **Reading Tip:** This file is completely standalone. You can download it and read it in any offline Markdown editor (like VS Code or Obsidian) in your favorite dark theme.

## Table of Contents

1. [Project Setup](#project-setup)
2. [Global API (PhotonClient)](#global-api-photonclient)
3. [Network Components](#network-components)
   * [PhotonView](#photonview)
   * [PhotonTransformView](#photontransformview)
   * [PhotonRigidbodyView](#photonrigidbodyview)
   * [PhotonAnimatorView](#photonanimatorview)
4. [RPC (Remote Procedure Calls)](#rpc-remote-procedure-calls)

## Project Setup

Before you begin, ensure the plugin is enabled and add your Photon credentials to the project settings.

1. Go to **Project -> Project Settings**.
2. Enable *Advanced Settings* if needed, and locate or create the following parameters:
   * `addons/photon/connection/app_id` (String): Your unique App ID from the Photon Dashboard.
   * `addons/photon/connection/app_version` (String): Your application version (default is `"1.0"`).

## Global API (`PhotonClient`)

`PhotonClient` is a Singleton responsible for connecting to servers, managing rooms, and handling the lifecycle of networked objects. It is accessible from any script as `PhotonClient`.

### Properties

* `offline_mode` **(bool)** — If `true`, the plugin will emulate the network. Ideal for testing game logic locally without an actual connection to Photon servers.

### Connection & Lobby

* `connect_to_server(app_id: String = "", app_version: String = "", region: String = "eu")`
  Initiates a connection to Photon servers. If arguments are empty, it uses values from Project Settings.

* `join_lobby() -> bool`
  Enters the main lobby (required to receive the room list).

* `get_state() -> int` / `get_ping() -> int`
  Returns the current state of the Photon state machine and the current ping.

* `set_offline_mode(offline: bool)` / `get_offline_mode() -> bool`
  Enables or disables offline mode.

### Room Management

* `create_room(room_name: String, max_players: int = 4, is_open: bool = true, is_visible: bool = true, custom_properties: Dictionary = {})`
  Creates a new room with the specified parameters.

* `join_room(room_name: String)`
  Joins a specific room by its name.

* `join_random_room()`
  Randomly joins any available open room.

* `join_or_create_room(room_name: String)`
  Smart join: joins the room if it exists, otherwise creates a new one.

* `leave_room()`
  Leaves the current room. Automatically cleans up local network objects.

* `get_room_list() -> Dictionary`
  Returns the list of available rooms (updates while in the lobby).

* `load_network_scene(scene_path: String)`
  Synchronously loads a scene for all players (can only be called by the Master Client).

### Players & Properties

* `get_local_player_id() -> int`
  Returns the ID of the local player.

* `is_master_client() -> bool`
  Checks if the current player is the host (Master Client).

* `set_master_client(player_id: int) -> bool` / `get_master_client_id() -> int`
  Transfers host privileges to another player and retrieves the current host's ID.

* `get_player_list() -> Dictionary`
  A dictionary containing all players currently in the room.

* `set_player_property(key: String, value: Variant)` / `get_player_property(player_id: int, key: String) -> Variant`
  Sets and reads Custom Properties for a specific player.

* `set_room_property(key: String, value: Variant)` / `get_room_property(key: String) -> Variant`
  Sets and reads Custom Properties for the current room.

### Instantiation (Spawning Objects)

* `instantiate(prefab_path: String, position: Vector3 = Vector3(), rotation: Vector3 = Vector3()) -> Node`
  Creates a local copy of a scene and broadcasts a network command to spawn it for other clients. The scene **must** contain a `PhotonView` node.

* `destroy(target_node: Node)`
  Destroys the object locally and across the network. You can only destroy objects that you own.

### Signals (`PhotonClient`)

| Signal | Description |
| ----- | ----- |
| `connected_to_master()` | Successfully connected to Photon servers. |
| `disconnected()` / `connection_error(error_code)` | Disconnected or a connection error occurred. |
| `lobby_joined()` | Successfully entered the lobby. |
| `room_list_updated(rooms: Dictionary)` | Room list updated (triggers while in the lobby). |
| `room_created(room_name)` / `room_joined(room_name)` | Room successfully created or joined. |
| `room_left()` / `room_failed(error_code, msg)` | Left the room or failed to join. |
| `player_joined(player_id)` / `player_left(player_id)` | Another player joined/left the room. |
| `master_client_switched(new_id, old_id)` | The Master Client has changed. |
| `network_scene_loaded(scene_path)` | A scene was loaded via `load_network_scene`. |
| `player_property_changed(player_id, key, value)` | A player's custom property was updated. |
| `room_property_changed(key, value)` | A room's custom property was updated. |

## Network Components

To synchronize scenes, you must add specific nodes to your prefabs (Godot scenes).

### `PhotonView`

**The base node for network identity.** Absolutely any object that needs to be synchronized over the network or receive RPCs must have a `PhotonView` in its node tree.

* **Properties:**
  * `view_id` **(int)** — The unique network identifier of the object.
  * `owner_id` **(int)** — The ID of the owning client.

* **Methods:**
  * `is_mine() -> bool` — Returns `true` if the local player is the owner of the object (used to separate input logic).
  * `transfer_ownership(new_player_id: int)` — Transfers control rights of the object to another player.
  * `photon_rpc(method_name: String, args: Array, target: int = RPC_OTHERS)` — Calls a function on other clients (see the RPC section).

### `PhotonTransformView`

Automatically synchronizes position, rotation, and scale (`Node2D` and `Node3D`). Uses smooth interpolation (Dead Reckoning) under the hood.

* **Properties:**
  * `sync_position` / `sync_rotation` / `sync_scale` **(bool)** — Defines exactly what is transmitted over the network.
  * `sync_rate` **(float)** — Data transmission frequency (times per second, default `20.0`).
  * `lerp_speed` **(float)** — Smoothing speed during interpolation on other clients (default `15.0`).

### `PhotonRigidbodyView`

A node for properly synchronizing physics objects (`RigidBody2D` and `RigidBody3D`).

* **Properties:**
  * `sync_linear_velocity` / `sync_angular_velocity` **(bool)** — Enables synchronization of physics forces/impulses in addition to transforms.
  * `sync_rate` / `lerp_speed` — Frequency and smoothing settings (similar to TransformView).

### `PhotonAnimatorView`

Synchronizes `AnimationTree` parameters over the network (e.g., BlendSpace for walking animations).

* **Properties:**
  * `anim_tree_path` **(NodePath)** — The path to the `AnimationTree` node.
  * `sync_parameters` **(Array of Strings)** — An array of parameter names (e.g., `["parameters/BlendSpace2D/blend_position"]`) to sync.

## RPC (Remote Procedure Calls)

RPCs allow you to execute a function on a script attached to a `PhotonView` across other players' computers.

### Usage Example:

```gdscript
extends Node3D

@onready var view = $PhotonView

func _process(delta):
    # Allow control only for the owner
    if view.is_mine():
        if Input.is_action_just_pressed("ui_accept"):
            # Call the "play_jump_effect" function on all OTHER clients
            view.photon_rpc("play_jump_effect", ["super_jump"], PhotonView.RPC_OTHERS)
            # Call it locally for ourselves
            play_jump_effect("super_jump")

# This function will be triggered over the network
func play_jump_effect(jump_type: String):
    print("Player jumped with style: ", jump_type)
    # Logic for playing particles/sounds goes here
```

### RPC Targets

When calling `photon_rpc`, the third argument determines who receives the message:

* `PhotonView.RPC_ALL` (0) — All players in the room (including the sender).
* `PhotonView.RPC_OTHERS` (1) — All players **except** the sender *(most commonly used)*.
* `PhotonView.RPC_MASTER_CLIENT` (2) — Only the host (Master Client) of the room.
* `PhotonView.RPC_ALL_BUFFERED` (3) — All players + players who join later (saved in the room cache).
* `PhotonView.RPC_OTHERS_BUFFERED` (4) — Other players + players who join later.

# Bye!!!
<img src="https://media1.giphy.com/media/v1.Y2lkPTc5MGI3NjExN2F2MnZpbzcwdXlrczBsc2p3MGZzYTN6bm5nYXJvZXl0NzZrbjZmaCZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/jVthNBwmRX50ceBGJd/giphy.gif" alt="ᗜˬᗜ" width="200" height="200">