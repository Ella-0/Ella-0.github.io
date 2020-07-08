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
2s (q_i q_k + q_j q_r) & 0 \\
2s (q_i q_j + q_k q_r) &
1 - 2s (q_i^2 + q_k^2) &
2s (q_j q_k - q_i q_r) & 0 \\
2s (q_i q_k - q_j q_r) &
2s (q_j q_k + q_i q_r) &
1 - 2s (q_i^2 + q_j^2) & 0 \\
0 & 0 & 0 & 1
\end{bmatrix}
$
