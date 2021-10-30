# Reverse Engineering Lunar

<p align="center">
  <img src="https://c.tenor.com/ZmEmeZQ4I1IAAAAd/lmao-lunar-client.gif">
</p>

### Introduction
Lunar Client, developed by Moonsworth, is a Minecraft modification that contains many modules and quality of life improvements; the ultimate goal of which is to make the game more enjoyable to play.

*This article is for education purposes only.*

### Motivation
Before I started working on reverse engineering Lunar, [Freddie](https://github.com/FreddieJLH) and myself had previous experience with reverse engineering from CheatBreaker.

One day I jokingly told Freddie that I'm going to work on Lunar, and from then on, I have been working on writing automated means of remapping the client.

I got particularly interested when I found out that [Vape](https://vape.gg/) can inject into Lunar, which was surpising to me since I thought they would have at least made some advance to prevent people from figuring out the Minecraft classes within the client.

### Contents
This article will cover a few distinct areas, respectively in order:
- Obfuscation & Deobfuscation
- Mixins
- WebSocket
- Intrusive Behaviour 

### Obfuscation
As Minecraft clients become more popular, many client developers will opt to keeping their Source Code private; This however does not stop people from decompiling their product and looking at the source code using a [Decompiler](https://en.wikipedia.org/wiki/Decompiler). This is why most clients these days use some means of [Obfuscation](https://en.wikipedia.org/wiki/Obfuscation) in order to make the code less readable, and sometimes more confusing.

What was immediately apparent in the case of lunar, is *Name Obfuscation*. Name obfuscation is where all of the classes, fields & methods are renamed, which makes reverse engineering a little tougher.

Upon further examination, I found that Lunar uses [Mixin](https://github.com/SpongePowered/Mixin) which immediately gave me hope, so I went over to their GitHub, and found their own fork on there. I began exploring and looking at the commits made by the Lunar development team.
The first thing I noticed, was they added a `@Proxy` annotation, which I became interested in.

Delving deeper into the JAR file I extracted after the client was launched, I found 3 interesting files, `patches.txt`, `mappings.txt` & `hashes.ini`.<br />
I took a look at their `VPatcher` and found that they use [Binary Patching](https://www.daemonology.net/bsdiff/) to apply patches to the game classes.<br />
I then found the exact library they were using, which was [jbsdiff](https://github.com/malensek/jbsdiff).

After further exploration, I found exactly how they patch the game classes, and recreated the behavior in a new project.

Here's a (simplified) breakdown of how their patching works:
- The lunar-prod file is loaded for the specified version
- The patcher finds the `mappings.txt` file within the production JAR.
- The patcher loads the vanilla version of Minecraft.
- The patcher applies the patches from `patches.txt` to the vanilla version.
- The patcher then runs the client with these applied patches.

These patches contain the merged Mixin members, as well as name obfuscation.

With the `mappings.txt` file I was able to extract all of the vanilla class names, which will later help me remove the name obfuscation present in the patched classes.

I matched an obfuscated class with a vanilla class and the overall structure was strikingly similar.

With this information, I was able to write a simple remapper which takes the original (*vanilla*) class and a patched class, and rename the methods based on their index in the class.

I quickly found a problem, Overwrites.
Overwrites are a mixin internal annotation that declares any method annotated with `@Overwrite` should be overwritten. By nature, Mixin puts these modified methods at the bottom of the class. My method of using Indexes would no longer be enough to reliably map the Minecraft method names.

Digging a little deeper, I found that if I lookup the Mixin parent class (found in the `@MixinMerged` annotation), I could find the original name of the Overwitten method. I thought I could use this name at first, but realised that lunar also obfuscates the `@Overwrite` method names in the patched class.

So, with that in mind I had to find another way to resolve the Overwrite targets. I found that merged Mixin members are added to the patched class in order, so I created a small algorythm to find the Overwrite's parent method in the Mixin class.

The algorythm can be found here:

```java
// index of the overwrite within the patched class.
int patchedOverwriteIndex = patched.methods.indexOf(patchedOverwrite);

// the index of the first method from the mixin class within this patched class.
int firstMemberIndex = -1;

int index = 0;
for (MethodNode method : patched.methods) {
    // get the mixin name from the @MixinMerged annotation.
    final String mixinName = getMixin(method);
    if (mixinName != null) {
        // if the mixin name of "method" is the same as the mixin name of the overwrite method.
        if (mixinName.equals(mixin.name)) {
            firstMemberIndex = index;
            break;
        }
    }
    index++;
}

// overwrite method index, pushed back by @Shadow methods.
int mixinMemberIndex = patchedOverwriteIndex - firstMemberIndex;

// skip constructor
int offset = 1;

// increase offset by 1 if the next method is a @Shadow method.
for (int i = 0; i < mixin.methods.size(); i++) {
    final MethodNode methodNode = mixin.methods.get(i);
    if ((methodNode.access & Opcodes.ACC_ABSTRACT) != 0) {
        offset++;
    } else if (!methodNode.name.endsWith("init>")) {
        break;
    }
}

// final index of the overwrite method within the mixin class.
mixinMemberIndex += offset;
```

This proved especially useful, because as mentioned the method in the mixin class retained it's original Minecraft method name. This way I could simply run through the Minecraft class, find a method with the matching name, and boom, I found what method the Overwrite belongs to. Then I could simply get the index of where the class lies in the Minecraft class, and re-arrange it in the patched class so my index-based mapping sollution will work.

Next on the list was the `@Proxy` annotation we breifly mentioned before.
This annotation retains the original name of the minecraft method, which was really kind of the Lunar developers to do. While `@Proxy` is similar to `@Overwrite`, it will keep the original method instead of directly overwriting it. This is sort of a method passthrough, if you want to call the original method you're overwriting you can't do thatby default with `@Overwrites` (unless I'm mistaken), so overwrites retain the original method for later use.

I saw a few classes were missing their original method, which was strange. Using the proxy method name, I could just run through the original minecraft class, find the method with the matching proxy target name, and insert the proxy method in place of where the original method would have been.

After all that work, I was safe to use the index-based mapping solution.
All Minecraft classes could now be read in full, and any other Mixin methods were adding back into their corresponding Mixin class.

### Mixins

The Mixins found in Lunar have an interesting twist; The client aims to work for all versions of Minecraft, which would require a lot of work on the developers side if they had to change every single module to work for the specified version.

The developers tackled this issue well, adopting what they call `MoonBridge`. 
This system consists of several hundred [Interface](https://docs.oracle.com/javase/tutorial/java/concepts/interface.html) classes for every Minecraft class they need to use in their client. These interfaces are implemented in the version-dependent Mixins.

This way, they would only need to modify the mixins in order for the client to work on a different version of Minecraft, instead of rewriting every individual component of the client itself.

### WebSocket

Lunar uses a websocket for many things, from retrieving your cosmetics, to forcibly crashing your client.

Let's have a look at how it works behind the scenes:
- You establish a connection with their authentication server.
- You complete a simple check to see if you're playing on a legitimate Minecraft account.
- You're given a token to establish a connection with the main socket.

Once you're connected to the main socket, you may send messages to other Lunar users, retrieve your owned cosmetics, check who else is using lunar (nametags) & other misc things.

Out of curiosity, [Pringles](https://github.com/PringlePot) and myself wrote a little program to mimic the client-side of the websocket. We found there is no rate-limits when sending messages, updating your status or the amount of friend request you can send.

Principally, it's almost identical to the CheatBreaker websocket, which can be found on my [GitHub](https://github.com/Decencies/CheatBreaker/tree/master/CheatBreaker/src/main/java/com/cheatbreaker/client/websocket).

### Intrusive Behaviour

Lunar have kept the websocket packets from CheatBreaker which allow them to forcibly crash your client by executing invalid instructions. Along with this, they have also kept the packets that allow them to view your process list without your notice.

In previous versions of lunar client, they also checked if you were proxying their services using the `hosts` file, which came as a suprise to me. I don't know what the main goal behind this was but it definitely didn't help much in the long run.


### Conclusion

Lunar client is a Minecraft client, with that in mind I think they have delivered a decent product.

I firmly belive that everyone should know how things work behind the scenes, hence I've written this article to cover how they've made it harder for people to fix their own copy of the software.
