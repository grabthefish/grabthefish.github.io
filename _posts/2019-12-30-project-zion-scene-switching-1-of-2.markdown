---
layout: post
title:  "Scene switching in Project Zion 1/2"
tags: godot devlog project-zion
date:   2019-12-30 15:00:00 +0100
---

A Godot game typically consists of multiple [scenes](https://docs.godotengine.org/en/3.1/getting_started/step_by_step/scenes_and_nodes.html). These scenes can contain multiple nodes and logic. You could for example have a scene for the main menu, for each town/map, for battles/fights, ...  
It would be useful to switch between these scenes how else are we going to show everyone our beautiful game?

What you could do I have a root scene and put all your scenes in there and hide/show the scenes accordingly. But there is a downside to this (probably multiple but I can only think of this one for now), and that is that all those scene get loaded in memory at the same time.  
Now if you have a tiny game like in the Godot [first game tutorial](https://docs.godotengine.org/en/3.1/getting_started/step_by_step/your_first_game.html) that is absolutly fine.  
But if your making a larger game with multiple cities, dungeons, ... it will be a lot to load in and process at once.

So what I did was make a GameRoot scene that loads and unloads scenes on request.
The scene looks like this at the moment:  
![GameRoot scene](/assets/2019-12-30-scene-switching-gameroot.jpg)

So a Scene must have a root Node, in this case I chose a [`Node2D`](https://docs.godotengine.org/en/3.1/classes/class_node2d.html) and named it GameRoot. Under the root node is another Node2D named CurrentScene, an [`AnimationPlayer`](https://docs.godotengine.org/en/3.1/classes/class_animationplayer.html) and [`CanvasLayer`](https://docs.godotengine.org/en/3.1/classes/class_canvaslayer.html) with a [`ColorRect`](https://docs.godotengine.org/en/3.1/classes/class_colorrect.html).
Lets look at the script behind the GameRoot:

{% highlight gdscript %}
extends Node2D

const START_SCENE = "res://scenes/game/mainmenu/MainMenu.tscn"

var nextScenePath

func _ready():
	# we don't need to fade out for the first scene because there is nothing to fade out of
	load_next_scene(START_SCENE, false)
	
func load_next_scene(path, fade_out = true):
	# save the path so we know which to load in before fading back in
	nextScenePath = path
	
	if fade_out:
		# before fading out we pause everything except the ColorRect and AnimationPlayer
		if $CurrentScene.get_child_count() > 0:
			SceneHelper.pause_all_nodes_except([$RectangleCL, $AnimationPlayer])
		
		# place the CanvasLayer before the CurrentScene node and play the fade-out animation
		$RectangleCL.set_layer(1)
		$AnimationPlayer.play("fade-out")
	else:
		# if we don't need to fade out just load the scene and fade in
		_switch_scene_and_fade_in()
	
func _switch_scene_and_fade_in():
	# load in the scene and create an instance
	var nextScene = load(nextScenePath).instance()
	
	# pause everything except the neccessary node for the animation
	if $CurrentScene.get_child_count() > 0:
		SceneHelper.pause_all_nodes_except([$RectangleCL, $AnimationPlayer])
	
	# delete the scene we are switching away from and add the new scene
	delete_children($CurrentScene)
	$CurrentScene.add_child(nextScene)
	
	# fade in the new scene
	$AnimationPlayer.play("fade-in")


func _on_AnimationPlayer_animation_finished(anim_name):
	if anim_name == "fade-out":
		# when the fade-out animation has finished and the screen
		# is completely black we start switching to the next scene
		_switch_scene_and_fade_in()
	if anim_name == "fade-in":
		# when the new scene is loaded in and visible we put the ColorRect
		# back behind the CurrentScene else it will catch every click
		$RectangleCL.set_layer(-1)
		SceneHelper.unpause_all()

# utility func for removing children from a node since that is not built in
func delete_children(node):
	for n in node.get_children():
		node.remove_child(n)
		n.queue_free()
{% endhighlight %}

The fade-in and fade-out animations are just simple color changes for the ColorRect from solid black to transparent:
![Animation track for fade-in](/assets/2019-12-30-scene-switching-animation.jpg)