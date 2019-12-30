---
layout: post
title:  "Scene switching in Project Zion 2/2"
tags: godot devlog project-zion
date:   2020-01-02 15:00:00 +0100
---

Now that we have some basic logic for switching scene in the GameRoot we need to be able to trigger it.

For that we will need to take a look at the scene for Ravenside and the target scene named House.  
![node structure for the Ravenside scene](/assets/2020-01-02-scene-switching-ravenside-nodes.jpg) ![node structure for the House scene](/assets/2020-01-02-scene-switching-house-nodes.jpg)  
For now we will only focus on the nodes under EntryPoints and SpawnPositions.

The entrypoint House is an [`Area2D`](https://docs.godotengine.org/en/3.1/classes/class_area2d.html) with a [`CollisionShape2D`](https://docs.godotengine.org/en/3.1/classes/class_collisionshape2d.html). The moment the Player enters the entrypoint it will trigger the code for scene switching.  
Now as you can see there is no signal hooked up to the House node, that's because it all gets done in the code behind in the `_ready` function. Let's take a look.

{% highlight gdscript %}
extends Node2D

func _ready():
	var areaHelper = AreaHelper.new(self, "res://assets/data/Ravenside/Ravenside.json")
	areaHelper.wire_area()
{% endhighlight %}

As you can see all the relevant code is in the `AreaHelper`, so this wasn't very helpful. Let's take a closer look at the `AreaHelper`.

{% highlight gdscript %}
extends Node

class_name AreaHelper

var scene
var data_file

func _init(area_scene, data_path):
	self.scene = area_scene
	self.data_file = data_path

func wire_area():
	var json = IO.load_text_file(data_file)
	var result = parse_json(json)
	
	if result.has("entryPoints"):
		_wire_entrypoints(result["entryPoints"])


# entrypoints
func _wire_entrypoints(entrypoints):
	for point in entrypoints:
		var node = point["node"]
		
		self.scene.get_node(node).connect("body_entered", self, "_handle_entrypoint_body_entered", [ point ])

func _handle_entrypoint_body_entered(body, entrypoint):
	# tilemaps also trigger the collision for the Area2D because its made of staticbodies (inherits from PhysicsBody2D)
	# to solve set the collision_mask en collision_layer to a different one than for the tilemaps
	# and put the mask and layer for the player on both so it still collides with the tilemap AND the area2D
	
	var target_scene_path = entrypoint["targetScene"]
	var target_scene = load(target_scene_path).instance()
	var position = target_scene.get_node(entrypoint["targetPositionNode"]).get_position()
	
	GlobalData.player.position = position
	
	SceneNavigation.goto_scene(target_scene_path)
{% endhighlight %}

So we instantiate the `AreaHelper` with the scene it needs to wire up and the path to the json data file with the config data.  
The json gets read and if the data contains an entryPoints property it gets processed. The json looks like this.  

{% highlight json %}
{
	"entryPoints": [
		{
			"node" : "EntryPoints/House",
			"targetScene" : "res://scenes/game/areas/Ravenside/House.tscn",
			"targetPositionNode" : "SpawnPositions/EntryPosition"
		}
	]
}
{% endhighlight %}

As you can see it's just an array of objects with 3 properties each:
* the `node` contains the nodepath to the entrypoint whose signal we need to connect to.
* the `targetScene` contains the path to the new scene we want to switch to.
* and finally the `targetPositionNode` contains the nodepath to the [`Position2D`](https://docs.godotengine.org/en/3.1/classes/class_position2d.html) node where the player needs to be put after switching scenes.
  
Now this might work perfectly for you but I had a little problem where everytime the scene for Ravenside got shown it immediately switched over to the House scene.  
If you've read the comment in the `_handle_entrypoint_body_entered` function from the `AreaHelper` script you already know why, if you haven't... why?  
Anyway there is a `TileMap` underneath my entrypoint, all the tiles in a tilemap are made out of [`StaticBody2D`](https://docs.godotengine.org/en/3.1/classes/class_staticbody2d.html) nodes which inherits from the [`PhysicsBody2D`](https://docs.godotengine.org/en/3.1/classes/class_physicsbody2d.html) node the signal we connected with reacts to, see ([`body_entered(PhysicsBody2D body)`](https://docs.godotengine.org/en/3.1/classes/class_area2d.html#class-area2d-signal-body-entered)).

There is a simple solution to this, change the collision layers and masks for the entrypoint and the player.
The layer and mask for the entrypoint get moved to a another one and those for the player get an extra one.  
![collision layer and mask for the entrypoint](/assets/2020-01-02-scene-switching-area2d-collision.jpg) ![collision layer and mask for the player](/assets/2020-01-02-scene-switching-player-collision.jpg)  
*entrypoint on the left and player on the right*

And thats pretty much all there is to my scene switching code.