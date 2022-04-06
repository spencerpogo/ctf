# Rev - wizardlike
Wizardlike - 500 points

Do you seek your destiny in these deplorable dungeons? If so, you may want to look elsewhere. Many have gone before you and honestly, they've cleared out the place of all monsters, ne'erdowells, bandits and every other sort of evil foe. The dungeons themselves have seen better days too. There's a lot of missing floors and key passages blocked off. You'd have to be a real wizard to make any progress in this sorry excuse for a dungeon! Download the [game](https://artifacts.picoctf.net/c/150/game). '`w`', '`a`', '`s`', '`d`' moves your character and '`Q`' quits. You'll need to improvise some wizardly abilities to find the flag in this dungeon crawl. '`.`' is floor, '`#`' are walls, '`<`' are stairs up to previous level, and '`>`' are stairs down to next level.

Hints:
- Different tools are better at different things. Ghidra is awesome at static analysis, but radare2 is amazing at debugging.
- With the right focus and preparation, you can teleport to anywhere on the map.

First thing, I open up the binary in Ghidra. The binary is stripped, but we can go to the `entry` function and double click on the first argument to `__libc_start_main` to get to the main function:
![[Pasted image 20220327161605.png]]

The main function seems to mention some external functions like `initscr`, `noecho`, `curs_set`, etc:
![[Pasted image 20220327162005.png]]

From `ldd` we can see that these symbols come from the `ncurses` library, a library for creating terminal user interfaces:
```
❯ ldd ./game
        linux-vdso.so.1 (0x00007ffce2588000)
        libncurses.so.6 => not found
        libtinfo.so.6 => not found
        libc.so.6 => /nix/store/4s21k8k7p1mfik0b33r2spq5hq7774k1-glibc-2.33-108/lib/libc.so.6 (0x00007fb6ab8f7000)
        /lib64/ld-linux-x86-64.so.2 => /nix/store/4s21k8k7p1mfik0b33r2spq5hq7774k1-glibc-2.33-108/lib64/ld-linux-x86-64.so.2 (0x00007fb6abaf4000)
```


