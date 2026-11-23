# Running CEDR on the VCU128 image (optional)

This guide cross-compiles [CEDR](https://github.com/UA-RCL/CEDR/tree/hcw-2024) (`hcw-2024` branch) and an API-based application for riscv64 on the host, copies them to the board over SSH, and runs them on the VCU128 Linux image.

It assumes the board is booted with Ethernet and SSH working (see [README.md](README.md)). The VCU128 design has no hardware accelerators, so CEDR runs everything on the CPU. (Support for accelerators is coming soon)

CEDR's own [README](https://github.com/UA-RCL/CEDR/blob/hcw-2024/README.md) and [GettingStarted.API.md](https://github.com/UA-RCL/CEDR/blob/hcw-2024/GettingStarted.API.md) describe the same flow for aarch64 (ZCU102). For the VCU128, use the riscv64 toolchain instead, as below.

## 1. Install the cross compiler

On the host (Debian/Ubuntu):

```
sudo apt install gcc-riscv64-linux-gnu g++-riscv64-linux-gnu cmake make
```

The CEDR toolchain file expects the compilers at `/usr/bin/riscv64-linux-gnu-gcc` and `/usr/bin/riscv64-linux-gnu-g++`, which is where this package installs them.

## 2. Clone CEDR

SSH:

```
git clone -b hcw-2024 git@github.com:UA-RCL/CEDR.git && cd CEDR
```

HTTPS:

```
git clone -b hcw-2024 https://github.com/UA-RCL/CEDR.git && cd CEDR
```

## 3. Cross-compile CEDR

From the CEDR root, use the riscv64 toolchain file in [toolchains](https://github.com/UA-RCL/CEDR/tree/hcw-2024/toolchains). It also points the linker at the prebuilt riscv64 libraries in `toolchains/riscv64-linux-gnu/lib/` (GSL, FFTW).

The build directory must be named `build_riscv`: the application Makefiles look for it under that name.

```
mkdir build_riscv && cd build_riscv && cmake --toolchain=../toolchains/riscv64-linux-gnu.toolchain.cmake .. && make -j$(nproc)
```

This builds `cedr`, `sub_dag`, `kill_daemon` and `libdash-rt/libdash-rt.so`. Check that they are RISC-V binaries:

```
file cedr
```

It should print `ELF 64-bit LSB ... UCB RISC-V`.

Unlike the ZCU102 build, leave out `-DLIBDASH_MODULES=...`: there are no FFT or GEMM accelerators in this design.

## 4. Cross-compile an application

The API applications in `applications/APIApps/` select the riscv64 cross compiler with `ARCH=riscv64`. For the radar correlator, from the CEDR root:

```
cd applications/APIApps/radar_correlator && make ARCH=riscv64
```

This produces `radar_correlator-riscv64.so`:

```
file radar_correlator-riscv64.so
```

Copy it, with its `input` folder, into the build directory, together with the daemon configuration:

```
cp -r radar_correlator-riscv64.so input ../../../build_riscv/ && cp ../../../daemon_config.json ../../../build_riscv/
```

## 5. Copy to the board

The board's file system is in RAM, so copy the files again after every boot. From the CEDR root:

```
scp -O -r build_riscv root@<board-ip>:/root/cedr
```

`-O` uses the classic SCP protocol; the board's SSH server (dropbear) has no SFTP server, which newer OpenSSH `scp` uses by default.

## 6. Run CEDR on the board

The default `daemon_config.json` runs 3 CPU worker threads with `Loosen Thread Permissions` set to `true`, which works on the single-core design.

Terminal 1, start CEDR:

```
ssh root@<board-ip>
```

```
cd /root/cedr && ./cedr -c ./daemon_config.json
```

Terminal 2, submit the application:

```
ssh root@<board-ip>
```

```
cd /root/cedr && ./sub_dag -a ./radar_correlator-riscv64.so -n 1
```

Terminal 1 shows the FFT tasks running on the CPU workers and ends with:

```
Radar correlator complete; lag Value is: -0.037000
```

Terminal 2, stop CEDR:

```
./kill_daemon
```

The timing log is in `log_dir/experiment0/`:

```
cat log_dir/experiment0/timing_trace.log
```

Other applications in `applications/APIApps/` (for example `pulse_doppler`, `SAR`, `lane_detection`, `wifi-tx/bin`) build the same way with `make ARCH=riscv64`; copy each `.so` and the files it reads into `build_riscv` before copying to the board.
