# The year of Linux on the desktop

## What we want
(Linux desktop is perfectly fine out of the box these days, with one
small caveat: until you seriously load it)
### Stability
Desktop must survive a serious bout of load without losing your
session or buggering it up so much that you have to reboot.
### Responsiveness
Desktop must stay at least minimally alive-feeling under load.
### Playing media without skipping
Your background music must not stutter, ever, full stop.

# System assumptions
## Your hardware is 12-ish years old or newer
(Sandy Bridge or better)
## 8G RAM
Otherwise you probably don't run a proper DE (by which I mean Gnome)
anyway.  6G _might_ be fine, 4G is severely borderline if you are from
planet Earth and have to use a browser.
## SSD
It's 2023 out there at the time of writing.
## Systemd, cgroupsV2 (unified), Gnome, Pipewire
I hear KDE is competent but never used it seriously; the rest is
non-mainstream with all that implies, go away and find you some
red-eyed friends on a stupid web forum or whatever.

# What we get out of the box
## Scheduling
Scheduler is not configured for desktop responsiveness: voluntary
preemption, big quantums, scheduling classes barely utilized,
swappiness value only appropriate for headless servers, etc.
## OOM handling
Depends on distro, but usually it's either just the kernel OOM killer
(impossible to disable, late to act, not exactly discerning --
basically you just don't want to ever get to the point where it is
invoked) or badly-configured systemd-oomd that may easily decide to
kill your Gnome Shell and will wait 20s or more till it does anything.
## Media
No threadirqs, no timer access for the `audio` group, no adding the
main user to `audio` at install time, no adding main user to `rtkit`
at install time (well, RTKit is not really needed with a recent
Pipewire at least, but still), no proper limit config to let your user
request realtime priorities at all, etc.  The "pro audio" web sites
lay the case out extensively and there are handy scripts to check your
configuration; some of the advice out there is of course optional (if
you don't really need any "pro" features) and/or outdated, but it's
still mostly very good.

At least recent Pipewire is (almost) fine out of the box (you _may_
have to increase headroom, especially for USB & BT audio, and I'm not
totally clear on whether you have to specify RT prio and niceness
explicitly or not, so I do anyway) -- provided all other configuration
is in order, which it is of course emphatically not.
## Zram swap
May not be enabled.
## MGLRU
At most just enabled (it's the kernel default when it exists at all,
probably?), with no further tuning (well, it's only in mainline since
6.1, so understandable).  Google people have put a ton of work into it
because they needed to get ChromeOS & Android to behave reasonably.
It's Good, you want to use it, probably on servers too.

# Theory of Getting There
## Stability
### Avoid kernel OOM killer
Configure systemd-oomd aggressively (there are alternatives to it,
like earlyoom etc., but I figure it's always better to go with
whatever paid professionals over at RedHat develop and support.  I
mean: it's documented(!), its logic is understandable, you have
runtime visibility into its configuration and cgroups, what's more to
ask).
### Avoid killing critical services on OOM
Systemd/Gnome take care to setup proper cgroups and slices, use them!
Basic idea is, only unleash preventive oomd on `app` and `background`
slices (if system/session services start eating RAM uncontrollably,
you have way bigger problems anyway IMO).  `machine` slice is matter
of use case and opinion, which I don't (yet) have.
## Responsiveness
### Swap to zram preferentially
(Happens by itself when you enable zram swap, but you still have to at
least do that).
### Enable and tune MGLRU
There's just one tuning knob (`min_ttl_ms`), set it to _something_.  I
use `1000`.
### Scheduling
Make sure (somehow, see below) that builds (and similar) get the
`batch` CPU scheduling class.

I/O scheduling depends on the file system; for example btrfs leaks I/O
priorities all over the place by design (or severe lack of incentive
to, same difference), so better avoid touching I/O scheduling at all
if you use btrfs.  You have an SSD anyway, so it's probably fine!
## Media
Apply the reasonable "pro audio" advice and you should be golden.

# Specifics
I use NixOS, so snippets below are meant to be pasted into your
`configuration.nix` or moral equivalent.  I have long forgotten how to
configure imperative distros (still rememeber how much it sucked
though).  I will not help you configure your imperative distro.

## Basics
```nix
boot.kernel.sysctl."vm.swappiness" = 1;
zramSwap.enable = true;

# tell Systemd to measure things (probably the default these days?
# doesn't hurt, anyway):
systemd = let
  accounting = ''
    DefaultCPUAccounting=yes
    DefaultMemoryAccounting=yes
    DefaultIOAccounting=yes
  '';
in {
  extraConfig = accounting;
  user.extraConfig = accounting;
  services."user@".serviceConfig.Delegate = true;
};
```

## Audio stutter prevention
```nix
boot.kernelParams = ["threadirqs"];
security.rtkit.enable = true;

# $username is you, whoever you are
users.users.${username}.extraGroups = ["audio" "rtkit"];
# allow members of "audio" to set RT priorities up to 90
security.pam.loginLimits = [{
  domain = "@audio";
  type = "-";
  item = "rtprio";
  value = "90";
}];
# expose important timers etc. to "audio"
services.udev.extraRules = ''
  DEVPATH=="/devices/virtual/misc/cpu_dma_latency", OWNER="root", GROUP="audio", MODE="0660"
  DEVPATH=="/devices/virtual/misc/hpet", OWNER="root", GROUP="audio", MODE="0660"
'';

# explicitly set Pipewire RT params (may not be necessary)
environment.etc."pipewire/pipewire.conf.d/99-custom.conf".text = ''
  context.modules = [
    { name = libpipewire-module-rt
      args = {
        nice.level = -11
        rt.prio = 19
      }
    }
  ]
'';

# increase output headroom.  this may make latency worse (not sure how
# noticeably) -- so if you game you may want to first try doing
# without it
environment.etc."wireplumber/main.lua.d/99-alsa-config.lua".text = ''
  -- prepend, otherwise the change-nothing stock config will match first:
  table.insert(alsa_monitor.rules, 1, {
    matches = {
      {
        -- Matches all sinks.
        { "node.name", "matches", "alsa_output.*" },
      },
    },
    apply_properties = {
      ["api.alsa.headroom"] = 1024,
    },
  })
'';
```

## Responsiveness tweaks (also help audio)
```nix
# use the handy system76-scheduler service (it is not in fact specific
# to System76 hardware, despite the name)
services.system76-scheduler = {
  enable = true;
  useStockConfig = false;  # our needs are modest
  settings = {
    # CFS profiles are switched between "default" and "responsive"
    # according to power source ("default" on battery, "responsive" on
    # wall power).  defaults are fine, except maybe this:
    cfsProfiles.default.preempt = "voluntary";
    # "voluntary" supposedly conserves battery but may also allow some
    # audio skips, so consider changing to "full"
  };
  processScheduler = {
    # Pipewire client priority boosting is not needed when all else is
    # configured properly, not to mention all the implied
    # second-guessing-the-kernel and priority inversions, so:
    pipewireBoost.enable = false;
    # I believe this exists solely for the placebo effect, so disable:
    foregroundBoost.enable = false;
  };
  assignments = {
    # confine builders / compilers / LSP servers etc. to the "batch"
    # scheduling class automagically.  add matchers to taste!
    batch = {
      class = "batch";
      matchers = [
        "bazel"
        "clangd"
        "rust-analyzer"
      ];
    };
  };
  # do not disturb adults:
  exceptions = [
    "include descends=\"schedtool\""
    "include descends=\"nice\""
    "include descends=\"chrt\""
    "include descends=\"taskset\""
    "include descends=\"ionice\""

    "schedtool"
    "nice"
    "chrt"
    "ionice"

    "dbus"
    "dbus-broker"
    "rtkit-daemon"
    "taskset"
    "systemd"
  ];
};
```

## Memory reclaim, OOM handling
```nix
# enable MGLRU.  change the min_ttl_ms value to taste
systemd.services."config-mglru" = {
  enable = true;
  after = ["basic.target"];
  wantedBy = ["sysinit.target"];
  script = let inherit (pkgs) coreutils; in ''
    ${coreutils}/bin/echo Y > /sys/kernel/mm/lru_gen/enabled
    ${coreutils}/bin/echo 1000 > /sys/kernel/mm/lru_gen/min_ttl_ms
  '';
};

# configure systemd-oomd properly
systemd.oomd = {
  enable = true;
  # disable the provided knobs -- they are too coarse, and also swap
  # monitoring seems like a bad idea, with btrfs anyway
  enableRootSlice = false;
  enableSystemSlice = false;
  enableUserServices = false;
  # change if 4s is too fast
  extraConfig.DefaultMemoryPressureDurationSec = "4s";
};
# kill off stuff if absolutely needed, limit to things killing which
# is unlikely to gimp system/desktop irreversibly, go only by PSI.
# tweak limits to taste, but be careful not to make them too high or
# you'll get the kernel OOM killer (on my machine 35% is too high, for
# example)
systemd.user.slices."app".sliceConfig = {
  ManagedOOMMemoryPressure = "kill";
  ManagedOOMMemoryPressureLimit = "16%";
};
systemd.slices."background".sliceConfig = {
  ManagedOOMMemoryPressure = "kill";
  ManagedOOMMemoryPressureLimit = "8%";
};
systemd.user.slices."background".sliceConfig = {
  ManagedOOMMemoryPressure = "kill";
  ManagedOOMMemoryPressureLimit = "8%";
};
```
