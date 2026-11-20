# SME Tech Eval & Demo: Prep Sheet (Linux + DevOps/SRE)

30-minute call. Suggested flow:

| Time | What |
|---|---|
| 0-3 min | Intro: 20+ years, freshers, classroom, 70% labs |
| 3-8 min | Your approach: day-wise flow, lab-first, how you pace freshers |
| 8-22 min | Live demo (one Linux, one DevOps) |
| 22-30 min | SME questions, your questions |

---

## 1. Linux (RHEL, 5 days)

### Demo A: LVM, online extend (best pick; covers day 4-5 ToC)

```bash
lsblk
pvcreate /dev/sdb
vgcreate vgdata /dev/sdb
lvcreate -n lvapp -L 2G vgdata
mkfs.xfs /dev/vgdata/lvapp
mkdir /app && mount /dev/vgdata/lvapp /app
df -hT /app

# grow while mounted
vgextend vgdata /dev/sdc
lvextend -r -L +2G /dev/vgdata/lvapp     # -r grows the filesystem too
df -hT /app

# make it persistent
blkid /dev/vgdata/lvapp                  # use UUID in /etc/fstab
mount -a                                 # validate fstab before reboot
```
Say: XFS can grow but cannot shrink; ext4 can do both (offline shrink).

### Demo B (if time): Permissions and ACL

```bash
chmod 2770 /shared            # SGID on directory: new files inherit group
chmod +t /shared              # sticky bit: only owner can delete own files
setfacl -m u:alice:rw /shared/report.txt
getfacl /shared/report.txt
umask
```

### Quick talking points for other ToC items

- **Patching:** `dnf check-update`, `dnf updateinfo list security`, `dnf update --security`, `dnf history undo <id>`, `needs-restarting -r`.
- **Logs:** `journalctl -u sshd --since "1 hour ago"`, `journalctl -p err -b`; persistent journal via `Storage=persistent` in `/etc/systemd/journald.conf`.
- **RHEL upgrade (Leapp):** `leapp preupgrade` → read the report, fix inhibitors → `leapp upgrade` → reboot → post-upgrade checks. Always take a snapshot/backup first.
- **Pacemaker:** concept plus command flow: `pcs host auth` → `pcs cluster setup` → `pcs cluster start --all` → configure STONITH → `pcs resource create vip ocf:heartbeat:IPaddr2 ip=...` → test with `pcs node standby`. Needs multi-VM labs, so say so.

### Likely SME questions (short answers)

- **Hard link vs soft link?** Hard link: same inode, same filesystem, survives deleting the original. Soft link: separate file holding a path, can cross filesystems, breaks if the target goes.
- **SUID / SGID / sticky?** SUID runs a file as its owner (e.g. `passwd`). SGID on a file runs as the group; on a directory, new files inherit the group. Sticky on a directory means only the owner (or root) can delete a file.
- **Process states?** R running, S sleeping, D uninterruptible (usually I/O), T stopped, Z zombie. A zombie is a finished child the parent hasn't reaped; you fix the parent, not the zombie.
- **kill -15 vs -9?** 15 (SIGTERM) asks for clean shutdown; 9 (SIGKILL) can't be caught, so no cleanup. Try 15 first.
- **Load average vs CPU usage?** Load is the average number of runnable plus uninterruptible tasks over 1, 5 and 15 minutes. Compare it with the number of cores.
- **Why LVM over plain partitions?** Resize online, span disks, snapshots.
- **rpm vs dnf?** rpm installs a single package and doesn't resolve dependencies; dnf resolves dependencies from repos.
- **Why fencing (STONITH)?** To prevent split-brain: a node that may be hung is forcibly powered off so two nodes never write shared data at once. Quorum is the related concept.
- **What does `/etc/fstab` do, and what if it's wrong?** Defines persistent mounts; a bad entry can drop the system to emergency mode, so always run `mount -a` first.

---

## 2. DevOps & SRE (5 days)

ToC order: Terraform (day 1) → DevOps overview and tools: Git, SonarQube, Jenkins, Artifactory, Aqua scan, GitHub Actions (days 2-3) → Docker and Kubernetes (days 3-5) → SRE (day 5).

### Demo A: CI/CD pipeline (best pick)

Show a Jenkinsfile (or a GitHub Actions workflow) with stages: checkout → build → unit test → (SonarQube scan) → docker build → image scan → push → deploy to Kubernetes. Explain what each stage protects against. Keep it to one running pipeline plus the file.

### Demo B: Kubernetes basics

```bash
kubectl create deployment web --image=nginx --replicas=3
kubectl expose deployment web --port=80 --type=NodePort
kubectl get pods,svc -o wide
kubectl set image deployment/web nginx=nginx:1.27       # rolling update
kubectl rollout status deployment/web
kubectl rollout undo deployment/web                     # rollback
kubectl scale deployment/web --replicas=5
```

### Demo C (optional): Terraform without a cloud account

Use the `local` provider or the Docker provider: `terraform init` → `plan` → `apply` → show `terraform.tfstate` → change a variable → `plan` shows the diff → `destroy`. Good for explaining state and plan vs apply on day 1.

### Tool gaps to prepare today

The ToC names tools that may not be strong points. If you haven't run them hands-on, spend 20-30 minutes today so you can talk about them honestly:

- **SonarQube:** `docker run -d -p 9000:9000 sonarqube:community`, scan a small project, show the quality gate.
- **Artifactory:** know what it is (binary/package/image repository, versioning, promotion between repos) and how Jenkins pushes to it.
- **Aqua scan:** image vulnerability scanning in the pipeline (shift-left security). If asked about your hands-on tool, say what you've used and how Aqua fits the same stage.
- **Terraform:** the demo above covers it.

If an SME asks about something you haven't done, say so plainly and describe how you would cover it in the lab. This is better than bluffing.

### Likely SME questions (short answers)

- **Why Terraform state?** Maps config to real resources, tracks metadata, enables diffs. Use remote state with locking for teams; never commit it (it can hold secrets).
- **`terraform plan` vs `apply`?** Plan shows the proposed changes without making them; apply executes them.
- **`count` vs `for_each`?** `count` is index-based; removing an item in the middle shifts the rest. `for_each` is keyed by name and is safer for sets of resources.
- **Provisioners?** `local-exec` and `remote-exec` are a last resort; prefer config management or images.
- **CI vs continuous delivery vs continuous deployment?** CI merges and tests often. Delivery keeps every build releasable with a manual approval. Deployment releases automatically.
- **Pod vs ReplicaSet vs Deployment?** Pod runs containers; ReplicaSet keeps N pods running; Deployment manages ReplicaSets and gives rolling updates and rollback.
- **Service types?** ClusterIP (internal), NodePort (node port), LoadBalancer (cloud LB). Labels and selectors connect a Service to Pods.
- **Control plane components?** API server, etcd, scheduler, controller manager; nodes run kubelet, kube-proxy and a container runtime.
- **SLI / SLO / SLA / error budget?** SLI is the measurement (e.g. latency, success rate), SLO the target, SLA the contract with consequences, error budget the allowed unreliability (100% minus SLO). If the budget is spent, slow releases and focus on reliability.
- **What is toil?** Manual, repetitive, automatable work that scales with service size; SRE aims to cap it and automate it.
- **Golden signals?** Latency, traffic, errors, saturation.
- **Blameless postmortem?** Focus on system and process causes, with action items, not on people.

---

## 3. Questions to ask the SMEs

1. Lab setup: provided by client or by you? How many VMs per participant (Pacemaker, Kubernetes)?
2. Versions: RHEL 8 or 9? Which Kubernetes distribution (minikube, kind, kubeadm)?
3. Batch size, and the freshers' background (CS grads, prior Linux exposure)?
4. For DevOps day 1-5: should Terraform target a cloud provider or run locally?
5. Daily timing and whether assessments are theory (MCQ) or hands-on.
