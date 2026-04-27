# Gardener Enhancements

This repository contains **Gardener Enhancement Proposals (GEPs)** - design and proposal documents for changes that are **relevant at the project-wide level** and require discussion and decision-making in Gardener's **Technical Steering Committee (TSC)**.

The structure and intent of this repository are inspired by [`kubernetes/enhancements`](https://github.com/kubernetes/enhancements).

---

## 📌 What Is a Gardener Enhancement?

A **Gardener Enhancement** is a proposal that meets the criteria for being discussed in [Gardener's steering meetings](https://gardener.cloud/community/steering/). These proposals typically cover **strategic, cross-cutting, high-impact, or high-risk topics** affecting the Gardener project as a whole.

Examples include:
- Major new features or architectural changes
- Significant changes to APIs or core behavior
- Topics with notable cross-team or ecosystem impact
- Initiatives requiring substantial development investment
- Proposals with elevated technical or operational risk

If a topic qualifies for [**Technical Steering**](https://gardener.cloud/community/steering/#%F0%9F%A7%91%E2%80%8D%F0%9F%92%BB-technical-steering), it belongs in this repository and must follow the **GEP process and template**.

---

## 🚫 Out of Scope

This repository **does not** host:
- Subproject-internal design documents
- Proposals that only affect a single Gardener subproject (e.g., MCM, etcd-druid) **and do not meet steering criteria**

Subprojects are free to maintain their own project-scoped proposal or design documents as long as the topic does **not** require steering-level discussion.
If it does, it must be submitted here as a GEP and presented in the relevant steering meeting.

---

## 🧭 Steering Committee

Gardener has one steering body:

- **Technical Steering Committee (TSC)**
  Focuses on *strategy, vision, prioritization, roadmap alignment, architecture, technical direction, and design decisions*.
  Discusses topics from the initial **”Why / Whether”** question through to the **”How”**.

Detailed information, criteria, and meeting processes can be found here:
👉 https://gardener.cloud/community/steering/

---

## ✅ Does My Topic Need a GEP?

A topic typically requires a GEP if it meets **any** of the following:

- Requires **significant development effort** (≈ 6+ person-months)
- Has **relevant cross-team impact**
- **Significantly impacts Gardener users or ecosystem**
- Introduces **high technical or operational risk**
- Introduces a **new Gardener extension** in the [Gardener GitHub organization](https://github.com/gardener)

If unsure, reach out to a steering committee member for guidance.

---

## 📝 How to Propose a Gardener Enhancement

> [!IMPORTANT]
> Please read through https://gardener.cloud/community/steering first.

1. **First circulate your idea to gauge community interest**
  — via Slack or community meetings
  - This helps ensure the proposal is well-motivated and increases the likelihood of a productive steering discussion.

2. **Create a new issue**
   - https://github.com/gardener/enhancements/issues/new?template=enhancement.md

3. **Create a new proposal file**
   - Copy the template from:
     [/geps/NNNN-gep-template](/geps/NNNN-gep-template/README.md)
   - Name it descriptively, e.g.
     `<issue-number>-node-specific-etcd-certificates.md`

4. **Add supporting assets (optional)**
   - Diagrams, slides, etc. in a matching subfolder

5. **Open a Pull Request**
   - Read https://gardener.cloud/community/steering/#%F0%9F%93%86-what-happens-after-opening-a-pr to learn what happens after opening the PR.
