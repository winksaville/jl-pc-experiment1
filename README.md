# PackageCompiler experiment 1

From: https://julialang.github.io/PackageCompiler.jl/dev/examples/plots/

First run julia and try to time plots, it fails:
```
ink@3900x:~
$ julia
               _
   _       _ _(_)_     |  Documentation: https://docs.julialang.org
  (_)     | (_) (_)    |
   _ _   _| |_  __ _   |  Type "?" for help, "]?" for Pkg help.
  | | | | | | |/ _` |  |
  | | |_| | | | (_| |  |  Version 1.5.2 (2020-09-23)
 _/ |\__'_|_|_|\__'_|  |
|__/                   |

julia> @time using Plots
ERROR: ArgumentError: Package Plots not found in current path:
- Run `import Pkg; Pkg.add("Plots")` to install the Plots package.

Stacktrace:
 [1] top-level scope at timing.jl:174
 [2] run_repl(::REPL.AbstractREPL, ::Any) at /build/julia/src/julia-1.5.2/usr/share/julia/stdlib/v1.5/REPL/src/REPL.jl:288
```

So we need to import Plots using the suggested Run `import Pkg; Pkg.add("Plots"):
```
julia> import Pkg; Pkg.add("Plots")
 Installing known registries into `~/.julia`
######################################################################## 100.0%
      Added registry `General` to `~/.julia/registries/General`
  Resolving package versions...
  Installed FFMPEG ────────────────────── v0.4.0
  Installed LibVPX_jll ────────────────── v1.9.0+1
  Installed Zlib_jll ──────────────────── v1.2.11+17
...
  [10745b16] + Statistics
  [8dfed614] + Test
  [cf7118a7] + UUIDs
  [4ec0a83e] + Unicode
   Building GR → `~/.julia/packages/GR/BwGt2/deps/build.log
```

Now use "Plots" and then plot, timing both and boy was that slow!!:
```
julia> @time using Plots
[ Info: Precompiling Plots [91a5bcdd-55d7-5caf-9e0b-520d859cae80]
 90.939379 seconds (11.92 M allocations: 729.846 MiB, 0.10% gc time)

julia> @time (p = plot(rand(5), rand(5)); display(p))
  4.669150 seconds (10.98 M allocations: 561.239 MiB, 1.71% gc time)
```

Next create precompile_plots.js:
```
wink@3900x:~/prgs/julia/projects/pc-experiment1
$ cat precompile_plots.jl
using Plots
p = plot(rand(5), rand(5))
display(p)
```

create sys_plots.so:
```
julia> using PackageCompiler
julia> create_sysimage(:Plots, sysimage_path="sys_plots.so", precompile_execution_file="precompile_plots.jl")
```

And now after precomiling:
```
wink@3900x:~/prgs/julia/projects/pc-experiment1
$ julia --sysimage sys_plots.so
               _
   _       _ _(_)_     |  Documentation: https://docs.julialang.org
  (_)     | (_) (_)    |
   _ _   _| |_  __ _   |  Type "?" for help, "]?" for Pkg help.
  | | | | | | |/ _` |  |
  | | |_| | | | (_| |  |  Version 1.5.2 (2020-09-23)
 _/ |\__'_|_|_|\__'_|  |
|__/                   |

julia> @time using Plots
  0.000713 seconds (320 allocations: 17.375 KiB)

julia> @time (p = plot(rand(5), rand(5)); display(p))
  0.509004 seconds (745.10 k allocations: 38.044 MiB, 0.75% gc time)
```

And now we see that loading our `sys_plots.so` is slightly slower:
```
wink@3900x:~/prgs/julia/projects/pc-experiment1
$ time julia --sysimage sys_plots.so -e ''

real	0m0.133s
user	0m0.075s
sys	0m0.041s
wink@3900x:~/prgs/julia/projects/pc-experiment1
$ time julia --sysimage sys_plots.so -e ''

real	0m0.158s
user	0m0.080s
sys	0m0.063s
wink@3900x:~/prgs/julia/projects/pc-experiment1
$ time julia --sysimage sys_plots.so -e ''

real	0m0.129s
user	0m0.080s
sys	0m0.035s
wink@3900x:~/prgs/julia/projects/pc-experiment1
$ time julia -e ''

real	0m0.122s
user	0m0.064s
sys	0m0.044s
wink@3900x:~/prgs/julia/projects/pc-experiment1
$ time julia -e ''

real	0m0.123s
user	0m0.063s
sys	0m0.046s
wink@3900x:~/prgs/julia/projects/pc-experiment1
$ time julia -e ''

real	0m0.090s
user	0m0.050s
sys	0m0.040s
wink@3900x:~/prgs/julia/projects/pc-experiment1
```

## Notes:

When I first tried to push this to github it complained
that sys_plots.so was to large and I should use git-lfs.github.com:
```
$ git push -u origin main
Enumerating objects: 6, done.
Counting objects: 100% (6/6), done.
Delta compression using up to 24 threads
Compressing objects: 100% (6/6), done.
Writing objects: 100% (6/6), 49.03 MiB | 5.45 MiB/s, done.
Total 6 (delta 0), reused 0 (delta 0), pack-reused 0
remote: error: GH001: Large files detected. You may want to try Git Large File Storage - https://git-lfs.github.com.
remote: error: Trace: c79dc214c6863e47fd82e8497b628349b252fba52b04610b1206597b13ff448e
remote: error: See http://git.io/iEPt8g for more information.
remote: error: File sys_plots.so is 217.54 MB; this exceeds GitHub's file size limit of 100.00 MB
To github.com:winksaville/jl-pc-experiment1.git
 ! [remote rejected] main -> main (pre-receive hook declined)
error: failed to push some refs to 'github.com:winksaville/jl-pc-experiment1.git'
```

So went to [git-lfs.github.com](https://git-lfs.github.com/) and followed
the instructions, except I installed using arch with `sudo pacman -Syu git-lfs`.
```
$ sudo pacman -Syu git-lfs
[sudo] password for wink: 
:: Synchronizing package databases...
 core                               130.9 KiB   935 KiB/s 00:00 [############################################] 100%
 extra is up to date
 community                            5.2 MiB  5.51 MiB/s 00:01 [############################################] 100%
:: Starting full system upgrade...
warning: freecad: local (0.18.16158-1) is newer than community (0.18.4-4)
resolving dependencies...
looking for conflicting packages...

Packages (2) iana-etc-20201012-1  git-lfs-2.12.0-2

Total Download Size:    3.96 MiB
Total Installed Size:  16.65 MiB
Net Upgrade Size:      12.75 MiB

:: Proceed with installation? [Y/n] Y
:: Retrieving packages...
 iana-etc-20201012-1-any            388.7 KiB  1039 KiB/s 00:00 [############################################] 100%
 git-lfs-2.12.0-2-x86_64              3.6 MiB  5.51 MiB/s 00:01 [############################################] 100%
(2/2) checking keys in keyring                                  [############################################] 100%
(2/2) checking package integrity                                [############################################] 100%
(2/2) loading package files                                     [############################################] 100%
(2/2) checking for file conflicts                               [############################################] 100%
(2/2) checking available disk space                             [############################################] 100%
:: Processing package changes...
(1/2) upgrading iana-etc                                        [############################################] 100%
(2/2) installing git-lfs                                        [############################################] 100%
:: Running post-transaction hooks...
```
Since my initial commit failed I used `git gui` to commit
.gitattributes and sys_plots.so by ammending the first commit.
And then I was able to successfully commit to github:
```
wink@3900x:~/prgs/julia/projects/pc-experiment1 (main)
$ git push origin main
Uploading LFS objects: 100% (1/1), 228 MB | 18 MB/s, done.                                                                       
Enumerating objects: 7, done.
Counting objects: 100% (7/7), done.
Delta compression using up to 24 threads
Compressing objects: 100% (6/6), done.
Writing objects: 100% (7/7), 3.11 KiB | 3.11 MiB/s, done.
Total 7 (delta 0), reused 0 (delta 0), pack-reused 0
To github.com:winksaville/jl-pc-experiment1.git
 * [new branch]      main -> main
```
