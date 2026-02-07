---
title: "Part 2: Ultimate Guide to Passing the CKAD Exam - Tips, Resources, and Strategies"
description: I cleared CKAD on my first attempt in just 14 days. Here are my tips, strategies, and resources for acing the exam.
author: Vaibhav Gagneja
date: 2024-06-04 16:15:40 +0000
categories: [DevOps, Kubernetes]
tags: [ckad, kubernetes, certification, devops, cloud-native]
image:
  path: https://cdn.hashnode.com/res/hashnode/image/upload/v1717427296142/c6bcfaa4-b36b-4afb-be83-1ecdce79ac8f.png
---

I cleared the Certified Kubernetes Application Developer (CKAD) exam on my first attempt. Despite CKAD being a comparatively tough exam, I managed to pass it in just 14 days. Here's how I did it.

## Exam Format

| Aspect | Details |
|--------|---------|
| **Duration** | 2 hours |
| **Format** | Performance-based, practical tasks in a command-line environment |
| **Environment** | Minimal Linux setup tailored for Kubernetes operations |
| **Questions** | Approximately 15-20 |
| **Passing Score** | 66% |
| **Proctoring** | Remotely proctored via audio, video, and screen sharing |

---

## Most Important Tips While Learning

- **Familiarize yourself with your editor** (vim, nano, emacs, etc.) to edit YAML files in a CLI environment.

- **Don't write YAML from scratch** during the exam. Use:
  ```bash
  --dry-run=client -o yaml > filename.yaml
  ```
  Or export a shortcut:
  ```bash
  export ds='--dry-run=client -o yaml > filename.yaml'
  ```

- **Rely on official documentation** for 80% of your learning. Bookmark these:
  - [kubernetes.io](https://kubernetes.io/)
  - [helm.sh](https://helm.sh/)

- **Master kubectl** - The most important resource: [kubectl commands reference](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#create)

- Take the course I mentioned in my [Part 1 article](/posts/ckad-certification-in-2-weeks/)

- **Understand the theory** - This helps you remember commands better by understanding the use case of each resource.

- **Practice is the key**

---

## Important Tips During the Exam

### Always check your context and namespace:
```bash
kubectl config current-context
kubectl config view --minify --output 'jsonpath={..namespace}'
```

### When stuck, use these commands:
```bash
kubectl explain --help                    # If you don't know how to use explain
kubectl explain deployments               # Get deployment structure
kubectl explain deployments.spec          # Expand to child properties
kubectl create deployment --help          # Get command help
```

### Learn JSONPath
If you have time, learn JSONPath - it's quite helpful! Free course: [KodeKloud JSONPath Quiz](https://kodekloud.com/courses/json-path-quiz/)

---

## Strategy During Learning

1. **Learn theory first**, then follow hands-on labs
2. **Don't dig too deep** into theory - know just enough
3. **Set time limits for practice**, but not for learning
4. **Use exam simulators wisely** - You get two simulators with the same questions. Don't panic if you score low - simulators are tougher than the real exam!

---

## Strategy During Exam

1. **This is an open-book exam** - reducing chances of getting stuck

2. **Stay calm** - Don't panic at all

3. **Skim through questions** quickly in 1-2 minutes first

4. **Prioritize**:
   - Easy questions first
   - Hard ones next
   - Flag difficult ones for last

5. **Save tough topics for last** - Dockerfile and older Kubernetes version questions

6. **Keep your environment clean** - No cheat sheets or notes, as a tidy space helps manage time effectively

---

## General Tips

- You can **reschedule the exam** as many times as you want until it expires
- Schedule on the **last date of validity** - if you don't pass, you get an extra 15 days for your second attempt
- **Read the official CKAD certification exam guides** carefully

> **These unique tips, if followed step by step, will help you ace the exam quickly.**

**All the best for the exam! :-)**

---

## Resources

| Resource | Link |
|----------|------|
| **Kubernetes Course** | [CKAD with Tests (Udemy)](https://www.udemy.com/course/certified-kubernetes-application-developer/) |
| **Exercises** | [CKAD-exercises (GitHub)](https://github.com/dgkanatsios/CKAD-exercises) |
| **More Practice** | [Medium - CKAD Practice Questions](https://medium.com/bb-tutorials-and-thoughts/practice-enough-with-these-questions-for-the-ckad-exam-2f42d1228552) |
| **Theory** | [Yumin Lee's Blog](https://yuminlee2.medium.com/) |
| **My Certificate** | [View Certificate](https://drive.google.com/file/d/1YkiZCcb00j6xNTDD9F9bQUk6EwWox0Ek/view) |
| **Simulator Questions** | [My Notes (Google Drive)](https://drive.google.com/file/d/1E_AJkTzPEd8kDiFRRbO7Swu_tc_j05Ys/view?usp=drive_link) |
