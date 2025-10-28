# Debugging-guide-ionic
how to set up debugging using curser/vscode with ionic cluster


# Complete Guide: Setting Up Remote Debugging for Your Codebase

This guide walks you through every single step to debug your code running on a GPU compute node from your laptop.

---

## Prerequisites
- A GPU compute node running your code
- SSH access to the cluster
- Your code already has the entry point (e.g., `linearprobe_gradients.py`) - what is an entry point??

---

## STEP 1: Get an Interactive GPU Node

### Using `salloc` (Interactive Allocation)
In the terminal once curser is connected to the login node,
```bash
# Request an interactive GPU node
salloc --account=visualai \
       --partition=<gpu_partition> \
       --gres=gpu:rtx_3090:1 \
       --cpus-per-task=8 \
       --mem=32G \
       --time=04:00:00

hostname
# Write down the hostname (e.g., "node019.ionic.cs.princeton.edu")
```
## STEP 2: Navigate to Your Project and Activate Conda Environment
On the compute node, run:
```
# Navigate to your project
cd /n/fs/dk-diffusion/repos/Unpaired-Multimodal-Learning/vision_language

# Activate your conda environment
source ~/.bashrc  # or wherever your conda is initialized
conda activate diffusion

# Verify it worked
which python
python --version
```

## STEP 3: Install `debugpy` in Your Environment

```bash
python -m pip install debugpy  # we already have this installed in diffusion
```

## STEP 4: Start Your Script with the Debug Server

Instead of running your script normally, you need to wrap it with debugpy:

```bash
# Basic format:
python -m debugpy --listen 0.0.0.0:5678 --wait-for-client your_script.py your_args

# For your specific case:
python -m debugpy --listen 0.0.0.0:5678 --wait-for-client linearprobe_gradients.py -c configs/linearprobe_gradient.yaml
```

### What This Does:
- `--listen 0.0.0.0:5678`: Makes debugpy listen on port 5678 (accessible from anywhere)
- `--wait-for-client`: **PAUSES** your script at the very beginning
- Your script waits until YOUR IDE connects

### What You Should See:
```
Waiting for client to connect...
```

**DO NOT CLOSE THIS TERMINAL!** Leave it running.

---

## STEP 5: Create SSH Tunnel from Your Laptop

### 1: Cursor Ports Tab (Laptop → Login Node)

When you SSH into the cluster, Cursor establishes an SSH connection to your **login node**.

When you go to the **Ports** tab and forward port 5678:
- Cursor automatically creates: `localhost:5678` on your laptop → `localhost:5678` on the **login node**
- This happens automatically! No command needed.
- The SSH connection is already established because you're editing files on the remote server.

### 2: Your Manual SSH Command (Login Node → Compute Node)

On the **login node**, run:
```bash
ssh -N -L 5678:node019.ionic.cs.princeton.edu:5678 node019.ionic.cs.princeton.edu
```

Let's break down this command:
```bash
ssh                                            # Start SSH
-N                                             # Don't open a shell, just forward ports
-L 5678:node019.ionic.cs.princeton.edu:5678   # Forward local port 5678 to remote
node019.ionic.cs.princeton.edu                 # Connect to the compute node
```

### What You Should See:
Nothing special - the command just sits there. That's correct! Leave it running.

---

## STEP 6: Configure Your IDE (Already Done!)

Your `.vscode/launch.json` is already configured:
```json
{
    "name": "Attach to remote debugpy",
    "type": "python",
    "request": "attach",
    "connect": { "host": "localhost", "port": 5678 },
    "justMyCode": false,
    "showReturnValue": true,
    "pathMappings": [
        {
            "localRoot": "${workspaceFolder}",
            "remoteRoot": "/n/fs/dk-diffusion/repos/Unpaired-Multimodal-Learning"
        }
    ]
}
```

---

## STEP 7: Set Breakpoints in Your Code

1. Open the file you want to debug: `linearprobe_gradients.py`
2. Find a line where you want execution to pause (e.g., line 34 inside a function)
3. Click in the gutter (left side of line numbers) to place a red dot
4. You can set multiple breakpoints

**Good places for breakpoints:**
- Line 460: `args = parser.parse_args(remaining_args)`
- Line 483: `args = parser.parse_args([], argparse.Namespace(**combination))`
- Line 34: Inside `train_with_gradient_saving` function
- Any line where you want to inspect variables

---

## STEP 8: Start Debugging

1. In VS Code/Cursor, go to the "Run and Debug" panel (Ctrl+Shift+D)
2. Select "Attach to remote debugpy" from the dropdown
3. Click the green play button (or press F5)
4. Watch the DEBUG CONSOLE at the bottom

### What Should Happen:
- The DEBUG CONSOLE should show: "Connected to debugpy"
- Your compute node terminal should show it's no longer waiting
- Your code starts running!

---

## STEP 9: When Your Breakpoint Hits

When your code reaches a breakpoint:
1. **Execution STOPS** at that line
2. The line will be highlighted in yellow
3. Check the **VARIABLES** panel on the left - you'll see all local variables
4. The **CALL STACK** shows the function hierarchy
5. You can hover over variables in your code to see their values

---

## STEP 10: Debug Controls (Bottom Toolbar)

Once paused at a breakpoint:

| Icon | Name | What It Does | Keyboard |
|------|------|--------------|----------|
| ▶️  | Continue | Resume execution until next breakpoint | F5 |
| ⏸️  | Pause | Pause execution (not usually needed) | Ctrl+Shift+Pause |
| ⏭️  | Step Over | Execute current line, don't go into functions | F10 |
| ⬇️  | Step Into | Go into function calls | F11 |
| ⬆️  | Step Out | Exit current function | Shift+F11 |
| 🔄  | Restart | Restart the debugger | Ctrl+Shift+F5 |
| ⏹️  | Stop | Stop debugging | Shift+F5 |

### Navigation Tips:
- **Continue (F5)**: Let code run until it hits the next breakpoint
- **Step Over (F10)**: Execute one line at a time, skip function internals
- **Step Into (F11)**: Go deeper into function calls
- **Step Out**: Finish current function and return to caller

---

## STEP 11: Inspecting Variables

### Variables Panel (Left Sidebar)
Shows all variables in the current scope:
- **Local**: Variables in the current function
- **Parameters**: Function arguments
- Expand objects to see their properties

### Watch Panel
Add expressions to monitor:
```python
len(image_loader)
model.parameters()
args.dataset
```

### Debug Console (Bottom Panel)
Execute Python code in the current execution context:
```python
>>> print(args)
>>> model.state_dict().keys()
>>> len(gradient_data)
```

---

## STEP 12: Understanding the "Call Stack"

The **CALL STACK** shows the chain of function calls:

Example:
```
1. train_with_gradient_saving()  ← You are HERE
2. main_with_gradients()
3. main()
4. <module> (the main block)
```

**How to use it:**
1. Click on different frames to see variables at different levels
2. Each frame shows variables that existed at that point in execution
3. This is why you might not see variables at first - you need to click the right frame!

---

## Common Issues and Solutions

### "Can't connect to debugpy"
- Check that SSH tunnel is running (step 5)
- Check compute node hostname is correct
- Try a different port (5679, 5680)
- Make sure firewall allows the port

### "Variables panel is empty"
- Click on different frames in the Call Stack
- Click on a frame that's in YOUR code (not library code)
- Use Continue (F5) to reach a breakpoint in your code

### "Breakpoints not working"
- Make sure the file path matches (check pathMappings in launch.json)
- Breakpoints might be in imported modules - check the Call Stack
- Set `"justMyCode": false` (already done)

### "Too many things happening"
- This is normal! Your code is running
- Set more breakpoints to pause at specific points
- Use Continue (F5) to jump to the next breakpoint

---

## Quick Reference Checklist

- [ ] SSH'd into cluster and got GPU node
- [ ] Written down compute node hostname
- [ ] Navigated to project directory
- [ ] Activated conda environment
- [ ] Installed debugpy
- [ ] Started script with debugpy (it's waiting)
- [ ] Created SSH tunnel from laptop
- [ ] Set breakpoints in code
- [ ] Started debugger in IDE
- [ ] Code connected and running
- [ ] Breakpoint hit, inspecting variables

---

## Example Workflow

1. **Setup (one time):**
   ```bash
   # Terminal 1: Compute node
   salloc --account=visualai --gres=gpu:rtx_3090:1 --time=4:00:00
   cd /n/fs/dk-diffusion/repos/Unpaired-Multimodal-Learning/vision_language
   conda activate my_env
   python -m pip install debugpy
   python -m debugpy --listen 0.0.0.0:5678 --wait-for-client linearprobe_gradients.py -c configs/linearprobe_gradient.yaml
   ```

2. **Tunnel (each session):**
   ```bash
   # Terminal 2: SSH tunnel
   ssh -N -L 5678:c1234:5678 -J your_user@login your_user@c1234
   ```

3. **Debug (in Cursor/VS Code):**
   - Set breakpoints
   - Press F5
   - Inspect variables
   - Use Continue/Step Over

---

## What "justMyCode: false" Means

- **true**: Only show YOUR code, skip library internals
- **false**: Show EVERYTHING, including libraries
- Setting to `false` helps you see what's happening in imported modules

---

## Understanding the Debug State

When you attach:
1. **Paused at entry**: Code hasn't run yet
2. **Press Continue (F5)**: Code starts running
3. **Breakpoint hit**: Execution stops, you can inspect
4. **Continue again**: Runs until next breakpoint
5. **Repeat**: Your debugging workflow!

---

## Next Steps After This Guide

Now that you're set up, you can:
- Add breakpoints wherever you want to inspect code
- Use Step Over/Into to navigate execution
- Watch variables change as code runs
- Use Debug Console to execute Python commands
- Modify variables on the fly (use with caution!)

Good luck debugging! 🐛🔍

