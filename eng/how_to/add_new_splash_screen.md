# SONIC-CLEAN-ENGINE-S.C.E.-TUTORIALS-

# How to add a new Splash Screen

- [Creating files](#creating-files)
	- [Splash.asm](#splashasm)
	- [Splash Screen Folder Tree](#splash-screen-folder-tree)
- [Registering the data](#registering-the-data)
- [Enabling the Splash Screen](#enabling-the-splash-screen)
	- [Changing the Initial Game Mode (Optional)](#changing-the-initial-game-mode-optional)

[← Back to previous page ](..)

# Creating files

First, let's create a `Splash Screen` folder in the [Screens](https://github.com/TheBlad768/Sonic-Clean-Engine-S.C.E.-/tree/Clone-Driver-v2/Screens) directory. Put all our new files in there.

```
📁 S.C.E. / S1-in-S3 (root)
└── 📁 Screens
    ├── 📁 Continue
    ├── 📁 Level
    ├── 📁 Level Select
    └── 📁 Splash Screen    <= CREATE ME 🥺
```

For the **Splash Screen**, you’ll need some data, specifically: _tile art, Enigma map, and palettes_. In this guide, we’re adding ready-made data; if you want to make your own picture, you will need to _create your own new files_ — that’s covered in the `"How to Build a Plane?"` guide.

For this guide, we'll grab ready-made data from the [Sonic-1-in-Sonic-3-S.C.E.-](https://www.google.com/search?q=https://github.com/TheBlad768/Sonic-1-in-Sonic-3-S.C.E.-/tree/flamedriver/Screens/Sega) source. 

Copy the `Enigma Map` and `KosinskiPM Art` directories from [Screens/Sega/S3K](https://github.com/TheBlad768/Sonic-1-in-Sonic-3-S.C.E.-/tree/flamedriver/Screens/Sega/S3K) (in the Sonic-1-in-Sonic-3-S.C.E.- source) into our `Splash Screen` folder:

```
📁 S1-in-S3 (root)
└── 📁 Screens
    └── 📁 Sega
        └── 📁 S3K
            ├── 📁 Enigma Map        <<= COPY THIS
            └── 📁 KosinskiPM Art    <<= COPY THIS
```

Copy the [Palettes](https://github.com/TheBlad768/Sonic-Clean-Engine-S.C.E.-/tree/Clone-Driver-v2/Screens/Level%20Select/Palettes) from [Screens/Level Select](https://github.com/TheBlad768/Sonic-Clean-Engine-S.C.E.-/tree/Clone-Driver-v2/Screens/Level%20Select), which you can find in the parent directory.

```
📁 S.C.E. / S1-in-S3 (root)
└── 📁 Screens
    └── 📁 Level Select
    └── 📁 SCE
        ├── 📁 KosinskiPM Art
        └── 📁 Palettes    <<= COPY THIS
```

## Splash.asm

Now, in our `Screens/Splash Screen` folder, let's create a text file named `Splash.asm` and paste this ready-to-go code into it:

```m68k
; ---------------------------------------------------------------------------
; Splash Screen
; ---------------------------------------------------------------------------

Splash_VDP:
		dc.w $8004								; disable HInt, HV counter, 8-color mode
		dc.w $8200+(VRAM_Plane_A_Name_Table>>10)				; set foreground nametable address
		dc.w $8300+(VRAM_Plane_B_Name_Table>>10)				; set window nametable address
		dc.w $8400+(VRAM_Plane_B_Name_Table>>13)				; set background nametable address
		dc.w $8700+(0<<4)							; set background color (line 3; color 0)
		dc.w $8B00								; full-screen horizontal and vertical scrolling
		dc.w $8C81								; set 40cell screen size, no interlacing, no s/h
		dc.w $9001								; 64x32 cell nametable area
		dc.w $9100								; set window H position at default
		dc.w $9200								; set window V position at default
		dc.w 0									; end marker

; =============== S U B R O U T I N E =======================================

SplashScreen:
		music	mus_Stop							; stop music
		jsr	(Clear_KosPlus_Module_Queue).w					; clear KosPlusM PLCs
		ResetDMAQueue								; clear DMA queue
		jsr	(Pal_FadeToBlack).w
		disableInts
		move.l	#VInt,(V_int_addr).w
		move.l	#HInt,(H_int_addr).w
		disableScreen
		jsr	(Clear_DisplayData).w
		lea	Splash_VDP(pc),a1
		jsr	(Load_VDP).w
		jsr	(Clear_Palette).w
		clearRAM Object_RAM, Object_RAM_end					; clear the object RAM
		clearRAM Lag_frame_count, Lag_frame_count_end				; clear variables
		clearRAM Camera_RAM, Camera_RAM_end					; clear the camera RAM
		clearRAM Oscillating_variables, Oscillating_variables_end		; clear variables

		; clear
		move.b	d0,(Water_full_screen_flag).w
		move.b	d0,(Water_flag).w
		move.b	d0,(HUD_RAM.status).w
		move.b	d0,(Update_HUD_timer).w						; clear time counter update flag
		move.w	d0,(Current_zone_and_act).w
		move.w	d0,(Apparent_zone_and_act).w
		move.b	d0,(Last_star_post_hit).w
		move.b	d0,(Debug_mode_flag).w

		; load main art
		QueueKosPlusModule	ArtKosPM_Splash, 0

		; load main mapping
		EniDecomp	MapEni_Splash, RAM_start, 0, 0, FALSE			; decompress Enigma mappings
		copyTilemap	VRAM_Plane_A_Name_Table, 320, 224

		; load main palette
		lea	(Pal_Splash).l,a1
		lea	(Target_palette).w,a2
		jsr	(PalLoad_Line16).w

		; set
		move.l	#VInt_Fade,(V_int_ptr).w					; set VInt pointer

.waitplc
		st	(V_int_flag).w							; set VInt flag
		jsr	(Process_KosPlus_Queue).w
		jsr	(Wait_VSync.skip).w
		jsr	(Process_KosPlus_Module_Queue).w
		tst.w	(KosPlus_modules_left).w
		bne.s	.waitplc							; wait for KosPlusM queue to clear

		; next
		move.l	#VInt_Main,(V_int_ptr).w					; set VInt pointer
		jsr	(Wait_VSync).w
		enableScreen
		jsr	(Pal_FadeFromBlack).w

		; set
		move.w	#3*60,(Demo_timer).w						; set to wait for 3 seconds

.loop
		jsr	(Wait_VSync).w

		; check exit
		tst.b	(Ctrl_1_pressed).w
		bmi.s	.exit								; if start was pressed, skip ahead
		tst.w	(Demo_timer).w
		bne.s	.loop

.exit

		; exit
		move.b	#GameModeID_LevelSelectScreen,(Game_mode).w			; set screen mode to Level Select (SCE)
		rts
```

## Splash Screen Folder Tree

Here is what the contents of your `Screens/Splash Screen` folder should look like:

```
📁 S.C.E. / S1-in-S3 (root)
└── 📁 Screens
    ├── 📁 Continue
    ├── 📁 Level
    ├── 📁 Level Select
    └── 📁 Splash Screen
        ├── 📁 Enigma Map
        │   └── 🗺️ Foreground.eni
		├── 📁 KosinskiPM Art
		│   └── 🖼️ Foreground.kospm
		├── 📁 Palettes
		│   └── 🎨 1.pal
		└── 📄 Splash.asm
```
# Registering the data

Now it's time to let our **Splash Screen** data "move in." In the [Data](https://github.com/TheBlad768/Sonic-Clean-Engine-S.C.E.-/tree/Clone-Driver-v2/Data) folder, you'll need to register these lines in the following files:

```
📁 S.C.E. / S1-in-S3 (корень)
└── 📁 Data
    ├── 📄 Enigma Data.asm
    ├── 📄 Kosinski Plus Module Data.asm
    └── 📄 Palette Data.asm
```

If you’re not sure where to "move these lines in," just drop them right before the **Level Select** lines.

- For tile art in [Kosinski Plus Module Data.asm](https://github.com/TheBlad768/Sonic-Clean-Engine-S.C.E.-/blob/Clone-Driver-v2/Data/Kosinski%20Plus%20Module%20Data.asm "Kosinski Plus Module Data.asm"):

```
; ===========================================================================
; Kosinski Plus Module compressed Splash screen graphics
; ===========================================================================

;		Attribute	| Filename	| Folder

		incfile.b	ArtKosPM_Splash, "Screens/Splash/KosinskiPM Art/Foreground.kospm"
```

- For Enigma map in [Enigma Data.asm](https://github.com/TheBlad768/Sonic-Clean-Engine-S.C.E.-/blob/Clone-Driver-v2/Data/Enigma%20Data.asm "Enigma Data.asm"):

```
; ===========================================================================
; Enigma compressed Splash screen data
; ===========================================================================

;		Attribute	| Filename	| Folder

		incfile.b	MapEni_Splash, "Screens/Splash/Enigma Map/Foreground.eni"
```

- For palettes in [Palette Data.asm](https://github.com/TheBlad768/Sonic-Clean-Engine-S.C.E.-/blob/Clone-Driver-v2/Data/Palette%20Data.asm "Palette Data.asm"):

```
; ===========================================================================
; Palette Splash screen data
; ===========================================================================

;		Attribute	| Filename	| Folder

		incfile.b	Pal_Splash, "Screens/Splash/Palettes/1.pal"
```
# Enabling the Splash Screen

All that's left is to enable the new **Game Mode**. Below are the instructions on how to do it, or you can check out the changes in the GitHub commit [here](https://github.com/Nichloya/Sonic-Clean-Engine-S.C.E.-Extended-/commit/7b0651c7c2229bdf41811d974231aafbace5bcb5).

```diff
+ Add the green lines
- Remove the red lines
```

Include our `Screens/Splash Screen/Splash` by adding it to the include list in [Engine/Includes.asm](https://github.com/TheBlad768/Sonic-Clean-Engine-S.C.E.-/blob/Clone-Driver-v2/Engine/Includes.asm):

```diff
 ; ---------------------------------------------------------------------------
 ; Objects data pointers
 ; ---------------------------------------------------------------------------

		include "Data/Objects Data.asm"

+; ---------------------------------------------------------------------------
+; Splash screen modules
+; ---------------------------------------------------------------------------
+
+		include "Screens/Splash/Splash.asm"
+
 ; ---------------------------------------------------------------------------
 ; Level Select screen modules
 ; ---------------------------------------------------------------------------

		include "Screens/Level Select/Level Select.asm"

```

In [Engine/Constants.asm](https://github.com/TheBlad768/Sonic-Clean-Engine-S.C.E.-/blob/Clone-Driver-v2/Engine/Constants.asm) add our new screen to the `Game mode routines` constant list:

```diff
 ; ---------------------------------------------------------------------------
 ; Game mode routines
 ; ---------------------------------------------------------------------------

 offset := Game_Modes
 ptrsize := 1
 idstart := 0

+GameModeID_SplashScreen =					id(GameMode_SplashScreen)			; 0
 GameModeID_LevelSelectScreen =					id(GameMode_LevelSelectScreen)			; 0
 GameModeID_LevelScreen =					id(GameMode_LevelScreen)			; 4
 GameModeID_ContinueScreen =					id(GameMode_ContinueScreen)			; 8

 GameModeFlag_TitleCard =					7						; flag bit
 GameModeID_TitleCard =						setBit(GameModeFlag_TitleCard)			; flag mask
```


Now, include the **Splash Screen** in the `Game mode routines` located in [Engine/Core/Security Startup 2.asm](https://github.com/TheBlad768/Sonic-Clean-Engine-S.C.E.-/blob/Clone-Driver-v2/Engine/Core/Security%20Startup%202.asm). Add it to the game mode list:

```diff
 ; ---------------------------------------------------------------------------
 ; Main game mode array
 ; ---------------------------------------------------------------------------

 Game_Modes:
+		GameModeEntry SplashScreen						; Splash mode
		GameModeEntry LevelSelectScreen					; Level Select mode (SCE)
		GameModeEntry LevelScreen						; Zone play mode
		GameModeEntry ContinueScreen					; Continue mode
```

The `GameModeEntry` macro uses the inserted variable to locate the screen, so the names must match exactly.

## Changing the Initial Game Mode (Optional)

If you want the game to start with the **Splash Screen** instead of the **Level Select**, change this line of code in [Engine/Core/Security Startup 2.asm](https://github.com/TheBlad768/Sonic-Clean-Engine-S.C.E.-/blob/Clone-Driver-v2/Engine/Core/Security%20Startup%202.asm)

```m68k
move.b	#GameModeID_LevelSelectScreen,(Game_mode).w
```

To this line:

```m68k
move.b	#GameModeID_SplashScreen,(Game_mode).w
```

Instead of **Level Select**, our new **Splash Screen** will now be the first thing to load.