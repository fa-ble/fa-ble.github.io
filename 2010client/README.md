30/9/2026 6:00 PM

How to make your revival have working (not secure) clients speedrun any %%%%%

Unlike the other tutorial where you were given a code, you won't get one here

Step 0:
  what you're gonna need or wanna know
  
  CreateGame(44340105256) -> creates game
  
  StartGame(any int) -> starts game

  ExecUrlScript -> execs a script from a url

  ExecScript -> execs a script from a file

  RobloxAuthenticate -> verification

Step 0.5:
  Do not patch:
    Trust Check
    Player Ids and Names
    Extranet

Step 1:

  Learn how URIs work to be able to launch RobloxApp.exe from the browser

Step 2:

  Quick and simple way to connect to a server:

    RobloxApp.exe -script dofile("link to join script")

  If you do this then you gotta patch extranet and it might allow people to self-patch a random 2010 client to hop onto your servers and exploit

Step 2B: 
  Learn how to use COM and ActiveX Objects
  
  How to get COM ActiveX Objects:
    Run RobloxApp.exe /regserver to expose ActiveX Objects (RobloxApp.App, IWorkspace)
    Create a launcher in C++ using Interop to interact with ActiveX since its deprecated

Step 3: 
  All ID for RobloxApp.App:CreateGame() explained
  
  44340105256 - Used for creating a game
  
  4569876 - Script
  
  4631452 - Script
  
  5689090 - Script
  
  4589421 - url
  
  5723392 - Url
  
  5623469 - Url

  Step 0.5:
    Use RBXGSConHost from the previous tutorial to make it run a host script instead of a render script
    Get a join script somewhere ok idc just make sure you do :CreateLocalPlayer(0), no number bigger
  
  Step 1:
    
    Requirements (BEFORE CONTINUING):
      
      A 10c subdomain that allows bot scraping (ct8.pl, small.pl)
      
      A 10c TLD (.com, .xyz, or a digitalplat TLD)
    
    RobloxApp.App.RobloxAuthenticate("http://domain.com/Login/Negotiate.ashx", "TICKET") <----- Negotiate.ashx must have a octet header and it must return false

  Step 2:
    
    $Game = RobloxApp.App:CreateGame(44340105256) <---- mandatory number, also exposes game stuff

  Step 3:

    workspace.ExecUrlScript(joinUrl)

    now where do you get workspace?????

    App::CreateGame(L"44340105256", null) the client returns an IWorkspace marshaled back as a VARIANT  with  vt=9 (VT_DISPATCH). That dispatch pointer lands is the COM workspace

  Step 4:

    You are done
