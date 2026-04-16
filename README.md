# Kubernetes Microservices Security Essentials - Demo Scripts

**Course:** Kubernetes Microservices Security Essentials (Video Course)  
**Author:** Chris Jackson, CCIEx2 #6256 — Distinguished Architect, Cisco  
**Publisher:** Pearson IT Certification  
**Published:** April 4, 2025  
**ISBN-10:** 0-13-544627-9  
**ISBN-13:** 978-0-13-544627-0  

This repository contains demo scripts and lab exercises from the **Kubernetes Microservices Security Essentials** video course. The course covers comprehensive security practices for protecting Kubernetes-based microservices — enforcing robust security policies, safeguarding sensitive data, implementing container isolation, and securing inter-pod communication with Cilium. It is part of the larger **Kubernetes Security Essentials (Video Collection)**.

For the complete CKS (Certified Kubernetes Security Specialist) course materials, see the [CKS Demo Scripts repository](https://github.com/chrijack/CKS-demo-scripts).

---

## Course Content

All lesson materials are located in the `lessons/` directory.

### Lesson 1: Use Appropriate Pod Security Standards

| Lesson | Topic | Link |
|--------|-------|------|
| 1.1 | Resource Isolation with ResourceQuota and LimitRange | [lessons/lesson-1.1/lesson-1.1.md](lessons/lesson-1.1/lesson-1.1.md) |
| 1.2 | Configure Security Contexts | [lessons/lesson-1.2/lesson-1.2.md](lessons/lesson-1.2/lesson-1.2.md) |
| 1.3 | Pod Security Admission | [lessons/lesson-1.3/lesson-1.3.md](lessons/lesson-1.3/lesson-1.3.md) |
| 1.4 | OPA Gatekeeper and ValidatingAdmissionPolicy | [lessons/lesson-1.4/lesson-1.4.md](lessons/lesson-1.4/lesson-1.4.md) |

### Lesson 2: Managing Kubernetes Secrets

| Lesson | Topic | Link |
|--------|-------|------|
| 2.1 | Understanding Kubernetes Secrets | [lessons/lesson-2.1/lesson-2.1.md](lessons/lesson-2.1/lesson-2.1.md) |
| 2.2 | Creating and Using Secrets | [lessons/lesson-2.2/lesson-2.2.md](lessons/lesson-2.2/lesson-2.2.md) |
| 2.3 | Using Secrets in Pods | [lessons/lesson-2.3/lesson-2.3.md](lessons/lesson-2.3/lesson-2.3.md) |
| 2.4 | Secrets Encryption at Rest | [lessons/lesson-2.4/lesson-2.4.md](lessons/lesson-2.4/lesson-2.4.md) |

### Lesson 3: Implement Container Isolation Techniques

| Lesson | Topic | Link |
|--------|-------|------|
| 3.1 | Containing Containers | [lessons/lesson-3.1/lesson-3.1.md](lessons/lesson-3.1/lesson-3.1.md) |
| 3.2 | Sandboxed Pods: Setup and Verification | [lessons/lesson-3.2/lesson-3.2.md](lessons/lesson-3.2/lesson-3.2.md) |
| 3.3 | Using gVisor | [lessons/lesson-3.3/lesson-3.3.md](lessons/lesson-3.3/lesson-3.3.md) |
| 3.4 | Using Kata Containers | [lessons/lesson-3.4/lesson-3.4.md](lessons/lesson-3.4/lesson-3.4.md) |

### Lesson 4: Implement Pod-to-Pod Encryption with Cilium

| Lesson | Topic | Link |
|--------|-------|------|
| 4.2 | Implementing Pod-to-Pod Encryption with Cilium | [lessons/lesson-4.2/lesson-4.2.md](lessons/lesson-4.2/lesson-4.2.md) |
| 4.3 | Implementing and Verifying mTLS with Cilium | [lessons/lesson-4.3/lesson-4.3.md](lessons/lesson-4.3/lesson-4.3.md) |

### Lesson 5: Secure Your Software Supply Chain

| Lesson | Topic | Link |
|--------|-------|------|
| 5.2 | ImagePolicyWebhook Policy Enforcement | [lessons/lesson-5.2/lesson-5.2.md](lessons/lesson-5.2/lesson-5.2.md) |
| 5.3 | Enforcing Software Supply Chain Security with Validating Admission Policy | [lessons/lesson-5.3/lesson-5.3.md](lessons/lesson-5.3/lesson-5.3.md) |
| 5.4 | Policy Enforcement: ImagePolicyWebhook | [lessons/lesson-5.4/lesson-5.4.md](lessons/lesson-5.4/lesson-5.4.md) |
| 5.5 | Policy Enforcement: Validating Admission Policy | [lessons/lesson-5.5/lesson-5.5.md](lessons/lesson-5.5/lesson-5.5.md) |

### Lesson 6: Scan Images for Known Vulnerabilities

| Lesson | Topic | Link |
|--------|-------|------|
| 6.2 | Container and Kubernetes Vulnerability Scanning with Trivy | [lessons/lesson-6.2/lesson-6.2.md](lessons/lesson-6.2/lesson-6.2.md) |
| 6.3 | Scanning with Trivy Operator | [lessons/lesson-6.3/lesson-6.3.md](lessons/lesson-6.3/lesson-6.3.md) |
| 6.4 | Using Kyverno, cosign, and trivy to validate SBOM and vulnerability attestation | [lessons/lesson-6.4/lesson-6.4.md](lessons/lesson-6.4/lesson-6.4.md) |

---

## Prerequisites

- Basic understanding of Kubernetes concepts and architecture
- Kubernetes cluster access (local or cloud-based)
- kubectl command-line tool
- Docker or container runtime knowledge
- Linux/Unix command-line familiarity

---

## Repository Structure

```
.
├── README.md
├── .gitignore
├── lessons/
│   ├── lesson-1.1/
│   ├── lesson-1.2/
│   ├── lesson-1.3/
│   ├── lesson-1.4/
│   ├── lesson-2.1/
│   ├── lesson-2.2/
│   ├── lesson-2.3/
│   ├── lesson-2.4/
│   ├── lesson-3.1/
│   ├── lesson-3.2/
│   ├── lesson-3.3/
│   ├── lesson-3.4/
│   ├── lesson-4.2/
│   ├── lesson-4.3/
│   ├── lesson-5.2/
│   ├── lesson-5.3/
│   ├── lesson-5.4/
│   ├── lesson-5.5/
│   ├── lesson-6.2/
│   ├── lesson-6.3/
│   └── lesson-6.4/
```

---

## Course Information

**Course URL:** https://www.pearsonitcertification.com/store/kubernetes-microservices-security-essentials-video-9780135446270

**Description:** This course covers comprehensive security practices for protecting Kubernetes-based microservices — enforcing robust security policies, safeguarding sensitive data, implementing container isolation, and securing inter-pod communication with Cilium. Part of the Kubernetes Security Essentials (Video Collection).

---

## License

MIT License

Copyright (c) 2025 Chris Jackson

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

**Attribution Required:** Demo scripts from the [Kubernetes Microservices Security Essentials (Video Course)](https://www.pearsonitcertification.com/store/kubernetes-microservices-security-essentials-video-9780135446270) by Chris Jackson, published by Pearson IT Certification (ISBN: 978-0-13-544627-0).

---

## Related Resources

- [CKS Demo Scripts Repository](https://github.com/chrijack/CKS-demo-scripts) - Complete Certified Kubernetes Security Specialist course materials
- [Kubernetes Security Essentials (Video Collection)](https://www.pearsonitcertification.com) - Full video collection by Pearson IT Certification
