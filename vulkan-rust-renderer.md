<script>
	MathJax = {
		tex: {
			inlineMath: [['$', '$']],
			processEscapes: true
		},
		svg: {
			fontCache: 'global'
		}
	};
</script>
<script src="https://polyfill.io/v3/polyfill.min.js?features=es6"></script>
<script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>

# Vulkan Rust Renderer

$\left\[
	\begin{matrix}
		a & b & c \\\
		d & e & f \\\
		g & h & i
	\end{matrix}
\right\]$

$
\begin{bmatrix}
1 - 2s (q_j^2 + q_k^2) &
2s (q_i q_j - q_k q_r) &
2s (q_i q_k + q_j q_r) & 0 \\\
2s (q_i q_j + q_k q_r) &
1 - 2s (q_i^2 + q_k^2) &
2s (q_j q_k - q_i q_r) & 0 \\\
2s (q_i q_k - q_j q_r) &
2s (q_j q_k + q_i q_r) &
1 - 2s (q_i^2 + q_j^2) & 0 \\\
0 & 0 & 0 & 1
\end{bmatrix}
$

However, this would require calulcating the length of the quaternion which is an expensive operation as it involves
a square root. We can fix this by only allowing a unit quaternion. This results in the following.

$
s = 1 \\\
\therefore \\\

\begin{bmatrix}
1 - 2(q_j^2 + q_k^2) &
2(q_i q_j - q_k q_r) &
2(q_i q_k + q_j q_r) & 0 \\\
2(q_i q_j + q_k q_r) &
1 - 2(q_i^2 + q_k^2) &
2(q_j q_k - q_i q_r) & 0 \\\
2(q_i q_k - q_j q_r) &
2(q_j q_k + q_i q_r) &
1 - 2(q_i^2 + q_j^2) & 0 \\\
0 & 0 & 0 & 1
\end{bmatrix}
$

This translates into the following method:

```rust

impl Quaternion {
	pub fn as_matrix(self) -> Mat4 {
		Mat4(
			1.0 - 2.0 * (self.v.1 * self.v.1 + self.v.2 * self.v.2),
			      2.0 * (self.v.0 * self.v.1 - self.v.2 * self.s),
			      2.0 * (self.v.0 * self.v.2 + self.v.1 * self.s),
			0.0,

			      2.0 * (self.v.0 * self.v.1 + self.v.2 * self.s),
			1.0 - 2.0 * (self.v.0 * self.v.0 + self.v.2 * self.v.2),
			      2.0 * (self.v.1 * self.v.2 - self.v.0 * self.s),
			0.0,
	
			      2.0 * (self.v.0 * self.v.2 - self.v.1 * self.s),
			      2.0 * (self.v.1 * self.v.2 + self.v.0 * self.s),
			1.0 - 2.0 * (self.v.0 * self.v.0 + self.v.1 * self.v.1),
			0.0,

			0.0,
			0.0,
			0.0,
			1.0,
		)
	}
}

```


