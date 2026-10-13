# Updating the VM to version 2.13: Role Overlap with the on-premise AI

Version 2.13 adds **Role Overlap**: the bank's own AI, Qwen on vLLM on this VM's GPU, compares each supervisor's activities with their direct reports', and colleagues' activities with each other. HR then reviews every case.

- **Nothing leaves the VM.** vLLM listens on 127.0.0.1 only and runs with internet use switched off.
- **Nothing in the portal's data is changed by the AI.** It only reads activities; its findings go to a separate review list.
- **2.13 contains everything in 2.12.** If 2.12 is not installed yet, install this zip instead. Then also do the one-time 2.12 steps in `UPDATE_TO_2.12_VM.md`, part "In the browser", including the HR sheet reference load.

The update has two parts:
- **Part A:** the portal code (about 10 minutes). The database, `secret.key`, the SSL certificate and the Python environment stay as they are, and no `pip install` is needed.
- **Part B:** starting vLLM as a service (about 20 minutes), then connecting the portal to it.

VM folder: `/data/opt/ops-portal`. Portal service: `ops-portal`. AI service: `vllm-qwen` (new). Run the steps one by one.

---

## What is new

- **Role Overlap**, a new tab in the menu.
  - Only the admin, and logins given the new **Role Overlap** tick (Admin → Users → Edit access), see it.
  - **Only the admin can run the AI.** Logins with the tick see the results and review the cases.
- **COSO Overview** (for the Group Head):
  - six figures: supervisors doing their team's work; single-report managers; near-identical manager and report; colleagues doing the same work in the same region; average supervisory share; overlapping work in FTE (indicative);
  - every function in one table, coloured by how much overlap was found, with HR's review progress;
  - **One-page PDF** and **Download Excel**.
- **By Function** (for HR):
  - **Supervisors and their teams:** "same as team" %, supervisory work % and flags.
  - **Colleagues doing the same work**, under the same manager or in the same unit under different managers. Each pair shows the region and daily volume. Same region is flagged; different regions show as regional cover; region not set is flagged for checking.
  - Clicking a row opens a **side-by-side comparison**: the AI's matches with their reasons, **Confirm / Not the same** per match, an HR status and note, and **Add as Recommendation**.
- **Review Tracker:** every reviewed case, with its status (Open, Under review, Confirmed, Dismissed), owner and note.
- **Admin → AI Server:**
  - the vLLM address and model name, with **Save and Test Connection**;
  - requests at once;
  - the "Doing team's work" threshold (default 50%).
- **The flags:**
  - *Doing team's work*: at least the threshold share of the supervisor's activities is also done by the team.
  - *Single report*: one direct report.
  - *Near-identical*: one report does 70% or more of the manager's activities.
  - *Mainly supervisory*: 60% or more supervisory work. This is the healthy case.

  The AI is told that checking, approving or authorising the team's work is **not** the same as doing it, so maker-checker roles are not flagged.
- **Overlapping work (FTE)** is indicative. It is based on each person's share of activities, and nobody counts for more than once.

How a run works: the admin clicks **Run Analysis** for one function or for all functions. It runs in the background, you can leave the page, and a progress bar shows how far it is. There is one AI request per supervisor (large teams are split), plus batches of likely colleague pairs. One function takes minutes; all of COSO takes about an hour. **Stop** ends a run; the earlier results stay. A new run of a function replaces that function's results, and HR's reviews are kept.

---

## Part A: install the portal update

### A1. Copy the zip to the VM

```
scp ops-portal-v2.13-ai-update.zip YOUR_VM_USER@hblassist.domestic.hbl.com:/tmp/
```

### A2. Back up first (do not skip)

```
sudo systemctl stop ops-portal
STAMP=$(date +%Y%m%d_%H%M)
sudo mkdir -p /data/opt/ops-portal/backups
sudo python3 -c "import sqlite3,sys; s=sqlite3.connect('/data/opt/ops-portal/backend/data/portal.db'); d=sqlite3.connect(sys.argv[1]); s.backup(d); d.close(); s.close(); print('database backed up')" /data/opt/ops-portal/backups/portal_before_2.13_$STAMP.db
sudo tar -czf /data/opt/ops-portal/backups/code_before_2.13_$STAMP.tgz -C /data/opt/ops-portal --exclude backend/venv --exclude backend/data --exclude backups .
sudo python3 -c "import sqlite3,glob; f=sorted(glob.glob('/data/opt/ops-portal/backups/portal_before_2.13_*.db'))[-1]; c=sqlite3.connect(f); print(f); print(c.execute('pragma integrity_check').fetchone()[0], c.execute('select count(*) from employees').fetchone()[0], 'employees', c.execute('select count(*) from work_items').fetchone()[0], 'activities')"
```

The last command should print `ok`, then your employee and activity counts. **Write the two numbers down.**

### A3. Install the new code

```
rm -rf /tmp/ops-update && python3 -m zipfile -e /tmp/ops-portal-v2.13-ai-update.zip /tmp/ops-update
ls /tmp/ops-update/ops-portal/backend/app/roleai.py && echo "zip OK"
sudo rsync -a --exclude 'backend/data' --exclude 'backend/venv' --exclude 'backups' --exclude 'wheels' /tmp/ops-update/ops-portal/ /data/opt/ops-portal/
```

If `rsync` is not installed, use this instead:

```
sudo rm -rf /data/opt/ops-portal/backend/app /data/opt/ops-portal/backend/static
sudo cp -r /tmp/ops-update/ops-portal/backend/app /tmp/ops-update/ops-portal/backend/static /data/opt/ops-portal/backend/
sudo cp -r /tmp/ops-update/ops-portal/deploy /data/opt/ops-portal/
```

### A4. Owner and permissions

```
sudo chown -R opsportal:opsportal /data/opt/ops-portal
sudo chmod 700 /data/opt/ops-portal/backend/data
sudo chmod 600 /data/opt/ops-portal/backend/data/portal.db /data/opt/ops-portal/backend/data/secret.key
sudo chmod -R a+rX /data/opt/ops-portal/backend/app /data/opt/ops-portal/backend/static
sudo chmod 755 /data/opt/ops-portal/backend/venv/bin/*
sudo restorecon -R /data/opt/ops-portal >/dev/null 2>&1 || true
```

### A5. Start the portal and check it

```
sudo systemctl start ops-portal
sleep 8
sudo systemctl status ops-portal --no-pager | head -5
sudo journalctl -u ops-portal -n 30 --no-pager
sudo -u opsportal python3 -c "import sqlite3; c=sqlite3.connect('/data/opt/ops-portal/backend/data/portal.db'); print(c.execute('select count(*) from employees').fetchone()[0], 'employees', c.execute('select count(*) from work_items').fetchone()[0], 'activities'); print('new tables:', [r[0] for r in c.execute(\"select name from sqlite_master where name like 'ro_%'\")])"
```

Check four things:
- the status shows **active (running)**;
- the log has no `Traceback`;
- the counts match step A2;
- `new tables` lists `ro_runs`, `ro_results` and `ro_cases`.

In the browser, press **Ctrl+F5**. **Role Overlap** is in the menu for the admin, and **Admin → AI Server** is there. The portal works normally even before the AI is set up.

---

## Part B: the on-premise AI (vLLM)

### B1. Find the four things the service needs

You need four values; write each one down.

**1. CUDA folder.** CUDA is installed in its own folder, so first find which one:

```
ls -d /usr/local/cuda* /opt/cuda* /data/opt/cuda* 2>/dev/null
```

Take the folder that has `bin/nvcc` and `lib64/libcudart.so*`, and check it:

```
CUDA=/the/cuda/folder        # put your folder here
ls $CUDA/bin/nvcc $CUDA/lib64/libcudart.so* && $CUDA/bin/nvcc --version | tail -2
```

**2. vLLM's Python.** This is the environment where vLLM is installed:

```
ls -d /data/opt/*/bin/python /opt/*/bin/python /home/*/*/bin/python 2>/dev/null
VPY=/the/vllm/env/bin/python  # put yours here
CUDA_HOME=$CUDA LD_LIBRARY_PATH=$CUDA/lib64 $VPY -c "import vllm, torch; print('vllm', vllm.__version__, '| torch', torch.__version__, '| GPU', torch.cuda.is_available(), torch.cuda.get_device_name(0) if torch.cuda.is_available() else '')"
```

It must print `GPU True` and the card's name. If it prints `GPU False`, the CUDA folder or the driver isn't right. Fix that first: vLLM will not start without the GPU.

**3. Model folder.** This is the folder with `config.json` and the `.safetensors` files of the quantised Qwen:

```
MODEL=/the/model/folder       # put yours here
ls $MODEL/config.json && grep -o '"quant_method": *"[a-z]*"' $MODEL/config.json
```

It should show `"quant_method": "awq"` or `"gptq"`. vLLM reads this by itself.

**4. User.** This is the Linux user who owns the model folder and the vLLM environment, usually your own login:

```
stat -c '%U' $MODEL $VPY
```

### B2. Try vLLM by hand first

Run this in a terminal and leave it running:

```
export CUDA_HOME=$CUDA CUDA_PATH=$CUDA PATH=$CUDA/bin:$PATH LD_LIBRARY_PATH=$CUDA/lib64 HF_HUB_OFFLINE=1 VLLM_NO_USAGE_STATS=1
$VPY -m vllm.entrypoints.openai.api_server --model $MODEL --served-model-name qwen \
  --host 127.0.0.1 --port 8000 --max-model-len 8192 --gpu-memory-utilization 0.90 --max-num-seqs 8
```

Wait until it prints `Uvicorn running on http://127.0.0.1:8000`; loading takes 1 to 5 minutes. Then, in a **second** terminal:

```
bash /data/opt/ops-portal/deploy/check_ai.sh
```

You should see the GPU, the model `qwen`, and a short JSON answer saying whether the two example activities are the same work. Then stop the first terminal with **Ctrl+C**.

If it fails here, see **Troubleshooting** below before going on.

### B3. Make it a service, so it starts with the VM

Put your four values in place of the placeholders and install the service:

```
sudo cp /data/opt/ops-portal/deploy/vllm-qwen.service /etc/systemd/system/vllm-qwen.service
sudo sed -i "s|VLLM_USER|$(stat -c '%U' $MODEL)|; s|CUDA_FOLDER|$CUDA|g; s|VLLM_PYTHON|$VPY|; s|MODEL_FOLDER|$MODEL|" /etc/systemd/system/vllm-qwen.service
grep -n "User=\|CUDA\|ExecStart\|--model" /etc/systemd/system/vllm-qwen.service
sudo systemctl daemon-reload
sudo systemctl enable --now vllm-qwen
sleep 120
sudo systemctl status vllm-qwen --no-pager | head -5
bash /data/opt/ops-portal/deploy/check_ai.sh
```

The `grep` line must show your real folders, with no capitalised placeholder left. `check_ai.sh` should answer as in B2.

To see what vLLM is doing: `sudo journalctl -u vllm-qwen -f`. To stop the AI without touching the portal: `sudo systemctl stop vllm-qwen`.

### B4. Connect the portal

1. **Admin → AI Server.** The address is `http://127.0.0.1:8000` and the model is `qwen`. Leave the API key blank. Click **Save and Test Connection**. It should say *Connected. Models: qwen*.
2. **Requests at once:** start with **4**. Raise it to 6 if the runs are slow and the GPU has room; lower it to 2 if you see time-outs.
3. **Role Overlap Rules:** "Doing team's work" from **50%**. Change it if HR prefers another level; it applies at the next run.

### B5. Give access

Go to **Admin → Users → Edit access** and tick **Role Overlap** for each HR login that should see it. The HR / executive preset ticks it. Only the admin can run the AI.

### B6. First run

1. **Role Overlap → By Function:** choose a small function and click **Run Analysis**. The progress bar moves; it should finish in a few minutes.
2. **Check the results:** open two or three supervisors. Do the matches make sense? Are the reasons sensible? Is checking or approving shown as **Supervisory** rather than matched?
3. **Run for everything:** when you are happy, go to **COSO Overview → Run Analysis (All Functions)**. It takes about an hour; you can leave the page, and the run continues on the VM.

Run it again after a big data change, such as a Review in Excel upload or a template round. Each run replaces that function's results, and HR's reviews stay.

---

## Troubleshooting

| What you see | What to do |
|---|---|
| `torch.cuda.is_available()` is `False` | The CUDA folder or the driver isn't right. Check `nvidia-smi` works, and that `LD_LIBRARY_PATH` points to the CUDA folder's `lib64`. |
| `libcudart.so... cannot open` | `LD_LIBRARY_PATH` doesn't include `CUDA_FOLDER/lib64`. Fix it in the service file, then `sudo systemctl daemon-reload && sudo systemctl restart vllm-qwen`. |
| `CUDA out of memory` when starting | In the service file, lower `--gpu-memory-utilization 0.90` to `0.85`, or `--max-model-len 8192` to `6144`. If it still fails, add `--enforce-eager`. Check nothing else is using the GPU (`nvidia-smi`). |
| vLLM says the model folder is not found or wants to download | `--model` must be the full folder path. `HF_HUB_OFFLINE=1` stops downloads. Check that `VLLM_USER` can read the folder. |
| Portal: *could not be reached* | `sudo systemctl status vllm-qwen`. vLLM must be running and the address must be `http://127.0.0.1:8000`. |
| Portal: *has no model called 'qwen'* | The `--served-model-name` in the service file and the model name in Admin → AI Server must be the same. |
| Portal: *answered 400 ... response_format* | Untick **Ask vLLM for JSON answers** in Admin → AI Server. This happens with older vLLM versions. |
| Run finishes with many *failed* requests | Usually time-outs. Raise **Wait per request** to 400, or lower **Requests at once** to 2. The run's message shows the first errors. |
| Run shows *the portal was restarted while this analysis was running* | The portal was restarted mid-run. Run it again. |

## Roll back to 2.12

```
sudo systemctl stop ops-portal
DB=$(ls -t /data/opt/ops-portal/backups/portal_before_2.13_*.db | head -1); CODE=$(ls -t /data/opt/ops-portal/backups/code_before_2.13_*.tgz | head -1); echo "$DB"; echo "$CODE"
sudo rm -rf /data/opt/ops-portal/backend/app /data/opt/ops-portal/backend/static
sudo tar -xzf "$CODE" -C /data/opt/ops-portal
sudo cp "$DB" /data/opt/ops-portal/backend/data/portal.db
sudo rm -f /data/opt/ops-portal/backend/data/portal.db-wal /data/opt/ops-portal/backend/data/portal.db-shm
sudo chown -R opsportal:opsportal /data/opt/ops-portal
sudo chmod 600 /data/opt/ops-portal/backend/data/portal.db
sudo restorecon -R /data/opt/ops-portal >/dev/null 2>&1 || true
sudo systemctl start ops-portal
```

The AI service is separate. Stop it if you no longer need it:

```
sudo systemctl disable --now vllm-qwen
```
