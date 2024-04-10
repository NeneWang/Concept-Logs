- Book Guide: https://jakevdp.github.io/PythonDataScienceHandbook/04.02-simple-scatter-plots.html
- How to use different figures to scatter plot? #card #[[Data Science]]
	- As the {{cloze third parameter}}
	- ![image.png](../assets/image_1712764189448_0.png)
	- ```python
	  rng = np.random.RandomState(0)
	  for marker in ['o', '.', ',', 'x', '+', 'v', '^', '<', '>', 's', 'd']:
	      plt.plot(rng.rand(5), rng.rand(5), marker,
	               label="marker='{0}'".format(marker))
	  plt.legend(numpoints=1)
	  plt.xlim(0, 1.8);
	  ```