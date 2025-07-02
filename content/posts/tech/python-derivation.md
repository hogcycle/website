---
date: '2025-07-02T14:44:10-04:00'
draft: false 
title: 'Building a Python applications without setup-tools in NixOS'
---

I have had NixOS deployed on all of my on-prem headless hardware for about 2 years now, and I still feel like I know nothing about it. 

Deploying someone elses module or installing someone elses package on NixOS is easy, but actually developing on it has quite the steep learning curve. This post is not intended to replace the actual documentation; it's just a polished version of my notes and a way for me to reinforce what I learned.

If you want to skip my long winded explanations, the complete derivation and service file can be found [here](https://github.com/hogcycle/discord-rpc-packaging)
## The App

The app I deployed is phin05's [discord-rich-presence-plex](https://github.com/phin05/discord-rich-presence-plex). It's written in Python and does not have an entry in Nixpkgs. Sure, it has a Dockerfile and a compose sheet, but whats the fun in that? 

## Building the derivation

This derivation is very simple. Nix expressions are just outputs based on inputs. The derivation takes these inputs:  

```nix
{
    lib,
    python3,
    fetchFromGitHub,
    makeWrapper
}:
```

We then use the [buildPythonApplication](https://nixos.org/manual/nixpkgs/stable/#buildpythonpackage-function) derivation wrapper included in the python3 input.
Note the ```rec``` keyword. This defines the set of options as a recursive set, meaning they can reference one another, such as how ```rev``` references ```version```.
This then defines the name and version of our application and uses the fetchFromGitHub function defined as an input earlier to grab the source code. 
```nix
python3.pkgs.buildPythonApplication rec { 
    pname = "discord-rich-presence-plex";
    version = "2.13.0"; 

    src = fetchFromGitHub {
        owner = "phin05";
        repo = "discord-rich-presence-plex"; 
        rev = "v${version}";
        sha256 = "sha256-dLCk7yeKw5NPgfm0N3jpWagY2F1vvh18ia3hgAOBaCE="; # use lib.fakeSha256 on first build to get the correct one.
```

Runtime and buildtime dependencies are defined within propogatedBuildInputs. 

```nix
    propagatedBuildInputs = with python3.pkgs; [
        plexapi
        requests
        websocket-client
        pyyaml
        pillow
    ];
```

Buildtime dependencies are defined within nativeBuildInputs. Note that these are Nix tools for the Nix buildtime, not dependencies for the Python application.  
```nix
    nativeBuildInputs = [
        makeWrapper
    ];
```


This project does not use setup-tools, so we will be skipping the build and testing process, as well as any dependency injection for those tests. 
```nix
    dontBuild = true; 
    format = "other"; 
    dontUseSetuptoolsBuild = true; 
    dontUseSetuptoolsCheck = true; 
    doCheck = false; 
```

Pretty self-explanatory. Builds the application and puts the appropriate files in the appropriate places. 
```nix
    installPhase = ''
        runHook preInstall
        
        mkdir -p $out/lib/discord-rich-presence-plex
        cp -r * $out/lib/discord-rich-presence-plex/
        
        mkdir -p $out/bin
        makeWrapper \
        ${python3}/bin/python \
        $out/bin/discord-rich-presence-plex \
        --add-flags "$out/lib/discord-rich-presence-plex/main.py" \
        --prefix PYTHONPATH : "$out/lib/discord-rich-presence-plex:$PYTHONPATH" \
        --set DRPP_NO_PIP_INSTALL "true" # application specific, removes call to pip in script

        runHook postInstall    
    '';
```

Meta information for the application.
```nix
    meta = {
        homepage = "https://github.com/phin05/discord-rich-presence-plex"; 
        license = lib.licenses.gpl3;
        description = "Displays your Plex status on Discord using Rich Presence";
        maintainers = with lib.maintainers; [ hogcycle ];
    };
}
```

With our derivation now complete, we can export it as a package and then import that into systemPackages on our configuration.nix.

```nix
environment.systemPackages = let discord-rich-presence-plex = pkgs.callPackage ./discord-rich-presence-plex.nix {};
in with pkgs; [
    discord-rich-presense-plex
    # more system packages
]; 
```

Next I will be writing a systemd service to manage this application as well as a Nix module for managing it cleanly within my configuration, but that will come as a seperate post. 


