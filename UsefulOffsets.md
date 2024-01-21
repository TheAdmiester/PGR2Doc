## Useful XBE file offsets
0x28345E = DisableLive (set to 01 to disable and grey out the Content Download/Xbox Live buttons on the main menu)\
0x283478 = NoSpaceText, used by CN/JP/KR languages - where screen titles such as "Kudos World Series" becomes "KudosWorldSeries" with alternating colours, turns them into regular strings\
0x28348C = Unknown flag, setting to 0 appears to disable car physics and cause glitchy driver models\
0x2834E4 = PresentInterval (division by 60 for framerate, e.g. the default 02 means 30fps - believe the physics don't scale to 60fps will run sped up)\
0x283504 = HideOverlays (set to 01 to disable HUD in races/showroom/replays)\
0x285518 = ShowStringNumber (set to 01 to replace all strings with "String Number %d", with their index in Frontend\StringList.ini)\
0x285519 = HighlighHardcodedStrings (set to 01 to highlight all hardcoded (i.e. not calculated from indexes) strings red)\
0x283595 = UseGarage (set to 00 to allow selecting of any car from any class, regardless of ownership, for Kudos Series events)\
