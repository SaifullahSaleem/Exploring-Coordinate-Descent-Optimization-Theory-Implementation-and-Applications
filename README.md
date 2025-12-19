## 📄 Project Overview

This research explores the theory, implementation, and practical applications of **Coordinate Descent (CD)** algorithms. While traditional methods like Gradient Descent update all variables simultaneously, Coordinate Descent optimizes one variable (or block of variables) at a time. This project bridges the gap between theoretical convergence properties and real-world performance in high-dimensional spaces.

### Key Highlights:

* **Theoretical Foundations:** Analysis of cyclic vs. randomized update strategies.
* **Performance Evaluation:** Comparison with Gradient Descent and Newton’s Method.
* **Regularization:** Integration with Lasso (L1) and Ridge (L2) for sparse variable selection.
* **Implementation:** Practical application on the **Wine Dataset** using Binary Logistic Regression.

---

## 🚀 Algorithmic Implementations

The project implements two primary variations of Coordinate Descent:

1. **Best Coordinate Descent (BCD):** At each iteration, the algorithm selects the coordinate with the largest gradient magnitude for the update.
2. **Random Coordinate Descent (RCD):** In each iteration, a coordinate is selected at random for the update, offering better scalability for massive datasets.

---

## 📊 Dataset & Preprocessing

The study utilizes the **Wine Dataset** (available via UCI Machine Learning Repository) focusing on chemical analysis to determine the origin of wines.

* **Binary Classification:** Filtered to include only samples from Class 1 and Class 2.
* **Normalization:** Features were mean-centered and scaled by their range to ensure uniform convergence.
* **Target Mapping:** Class labels transformed to \{0, 1\} for logistic regression compatibility.

---

## 📈 Results & Findings

* **Accuracy:** Both Best and Random Coordinate Descent achieved **100% accuracy** on the Wine dataset.
* **Efficiency:** The algorithms showed rapid convergence to global minima for convex objectives.
* **Scalability:** Randomized CD proved highly effective at reducing computational overhead per iteration compared to full gradient methods.

| Feature | Coordinate Descent | Gradient Descent | Newton's Method |
| --- | --- | --- | --- |
| **Update Rule** | Single variable | All variables | All variables |
| **Complexity** | Low per iteration | Medium | High (Hessian) |
| **Best For** | High-dimensional/Sparse | General purpose | Small-scale/Exact |

---

## 🛠️ Usage

To run the implementations included in this repository:

1. **Clone the repository:**
```bash
git clone https://github.com/SaifullahSaleem/GIKI-Chatbot.git

```



2. **Run the analysis:**
```bash
python DS_221_CEP_code.ipynb

```



---

## 📚 References

* Johnson, R., & Zhang, T. (2021). *Accelerating stochastic coordinate descent for linear SVM.* JMLR.
* Liu, R., et al. (2021). *Stochastic coordinate descent for deep learning.* arXiv.
* Wang, M., & Yin, W. (2021). *Non-asymptotic convergence rate of coordinate descent algorithms.* JMLR.

---

Would you like me to help you format the **Table of Contents** with clickable links for this README?
