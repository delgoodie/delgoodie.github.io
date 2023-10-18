---
sortIndex: 1
sidebar: ue4guide
---

|                     |                                                                                                                                                                                                                                                                    |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Game Instance:      | All the entry point functions for on joining/finding/creating sessions/network failures (the functions here wrap the gamesession calls with logic on how it relates to the rest of the app like showing UI, etc). Also handles app suspention/app resume/app start |
| GameMode:           | Set the game match's state machine + rule enforcement.                                                                                                                                                                                                             |
| GameState:          | Random shizzit about gamestate that needs to be replicated to other clients like team scores                                                                                                                                                                       |
| PlayerState:        | Random shizzit about players that need to be replicated like current kills, suicides, teamnumber, team color                                                                                                                                                       |
| GameSession:        | Actual functions that manage game sessions (join, search, host) with online subsystem                                                                                                                                                                              |
| GameUserSettings:   | Settings for the app (sound, graphics, resolution, etc)                                                                                                                                                                                                            |
| LocalPlayer:        | Custom player input binds (this is where debug binds go, player custom remappings, etc)                                                                                                                                                                            |
| PlayerInput:        | Custom player input binds (this is where debug binds go, player custom remappings, etc)                                                                                                                                                                            |
| OnlineGameSettings: | Define properties about sessions that can be searched (like isLan, maxnumplayers, advertise, etc)                                                                                                                                                                  |
