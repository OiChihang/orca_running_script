# Concurrent ORCA Runner

`run_orca.sh` runs several independent ORCA quantum chemistry calculations
concurrently. It uses a FIFO semaphore and is compatible with older systems,
including GNU Bash 4.2. It does not require `wait -n` or newer Bash features.

Each ORCA input is assembled as:

```text
H
coordinates for one structure
T
```

The script processes an inclusive range of structures from a concatenated XYZ
coordinate file. At most `k` ORCA calculations run at the same time.

## Requirements

- GNU Bash 4.2 or later
- ORCA installed and executable
- Standard Unix tools: `awk`, `grep`, `mkfifo`, `mktemp`, `cat`, and `rm`
- A working directory containing `run_orca.sh`, `H`, and `T`

## Files

### `H`

The ORCA input placed before the coordinates. It normally contains the method,
basis set, parallel settings, memory settings, charge, and multiplicity.

Example:

```text
! METHOD BASIS TightSCF
%pal nprocs 4 end
%maxcore 4000
* xyz 0 1
```

### `T`

The ORCA input placed after the coordinates. For a normal XYZ geometry block:

```text
*
```

### Coordinate file

The coordinate file must contain consecutive XYZ structures. Every structure
must have exactly `natom` atoms and occupy `natom + 2` lines:

```text
number of atoms
comment line
element  x  y  z
element  x  y  z
...
```

The structures are numbered from `1`. Extra blank lines between structures are
not supported by the fixed-block extraction method.

## Configuration

Edit the settings near the beginning of `run_orca.sh` before running it.

### ORCA paths

Set the ORCA executable and required library paths:

```bash
ORCA="/path/to/orca/orca"

export PATH="/path/to/orca:$PATH"
export LD_LIBRARY_PATH="/path/to/orca/lib:${LD_LIBRARY_PATH:-}"
```

`ORCA` should point to the actual ORCA executable. Do not point it to a wrapper
that submits or backgrounds the calculation and then exits.

### Main parameters

```bash
natom=8       # atoms in each structure
k=4           # maximum concurrent ORCA calculations
pause=5       # seconds between launches
stop_grace=10 # seconds before force-killing a stopped calculation
```

- Set `natom` to the atom count in every XYZ structure.
- Set `k` according to the CPUs and memory available on the node.
- `k` must be at least `1`.
- `pause` reduces the chance of several large jobs starting at exactly the same
  time. It does not make the calculations sequential.

### CPU and memory check

`k` limits the number of ORCA calculations, not the number of cores used by
each calculation. Check `%pal nprocs` and `%maxcore` in `H` before running.

Approximate maximum core use:

```text
k x nprocs
```

Approximate requested memory:

```text
k x nprocs x maxcore
```

These totals must fit within the resources allocated to the current node.
Reducing only `k` may not be sufficient if each ORCA calculation requests many
cores or a large amount of memory.

## Usage

Make the script executable once:

```bash
chmod +x run_orca.sh
```

Run it with a coordinate file, a starting structure number, and an ending
structure number:

```bash
./run_orca.sh [filename] [startnumber] [endnumber]
```

Example:

```bash
./run_orca.sh structures.xyz 3 12
```

This processes structures `3` through `12`, inclusive.

## Output

For structure number `N`, the script creates:

```text
N.inp  ORCA input
N.out  ORCA output
```

The temporary coordinate file `N.tmp` is removed after `N.inp` is assembled.
The cleanup section also removes configured ORCA auxiliary files. Review its
file patterns before running, especially if restart or analysis files are
needed later.

## Completion checks

A calculation is reported as successful only when:

1. the ORCA process returns exit status `0`; and
2. its output contains `ORCA TERMINATED NORMALLY`.

For a multi-step ORCA input, one semaphore token represents the entire ORCA
invocation. The token is returned only after the ORCA process and all steps in
that input have finished.

Normal ORCA termination does not always mean that a requested scientific goal
was reached. For example, a geometry optimization can terminate normally after
reaching its maximum cycle count. Inspect the output for method-specific
convergence messages when needed.

## Interruption and errors

If input generation fails, a structure is missing, or the script receives
`HUP`, `INT`, or `TERM`, it stops its active workers and waits for them. ORCA is
first sent `TERM`; after `stop_grace` seconds it may be force-killed.

The main `.inp` and `.out` files are retained for inspection.

## Notes

- Run separate batches in separate directories to avoid filename collisions.
- Do not run two copies of the script in the same directory with overlapping
  structure numbers.
- Check available disk space before starting a large batch.
- Keep wavefunction or restart files when a later calculation depends on them.

## Contribution

OpenAI Codex assisted with reviewing the Bash 4.2 semaphore logic, identifying
error-handling and cleanup considerations, and preparing this English README.
