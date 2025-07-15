
**Step 1: Prepare codeword error index vector (x)**

$$

\begin{align*}

x[\,] &= \{ 1, 2, \dots, \text{max\_correctable\_cw\_symbol\_errors} \},\, where\\

max\_correctable\_cw\_symbol\_errors &= 15\,in\,case\,of\,RS\_544\\

\end{align*}

$$


For each index $i$ in vector $x$, $\text{codeword\_errors[i]}$ represents number of codewords with $i$ symbol errors i.e SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S$i$.

**Step 2: Compute logarithm codeword error ratio vector (y)**

By applying $log_{10}$ scaling to the codeword error ratio, we convert a non-linear codeword error decay (codeword_errors) 
into a linear trend that can be modeled using linear regression.

For each index $i$ in vector $x$, compute logarithm of codeword error ratio $y[i]$ as follows

$$

\begin{align*}

y_i &= \log_{10} \left( \frac{CW_i}{S_{CW}} \right)\,

where,\\[2em]

CW_i &= codewords\,with\,i\,symbol\,errors\\[2em]

S_{CW} &= \sum_{i=0}^{15} CW_i

\end{align*}

$$


**Step 3: Perform linear regresion to arrive at slope and intercept**

$$

\begin{align*}

\text{slope} &= \frac{n \sum xy - \sum x \sum y}{n \sum x^2 - (\sum x)^2} \\[2em]

\text{intercept} &= \frac{\sum y - \text{slope} \sum x}{n} \\[2em]

where, n &= number\,of\,data\,points\\

\text{This gives the best-fit line}\,, y &= slope \times x + intercept.
\end{align*}



$$

  

**Step 4: Compute extrapolated CER**

Using linear regression line, predicted CER for an index representing $j$ symbol errors is,

$$

\begin{align*}

\text{predicted\_cer}_j &= 10^{j \times \text{slope} + \text{intercept}}\\[1em]

\end{align*}

$$

Predicted cer for a window of codewords having uncorrectable symbol errors is

$$

\begin{align*}

\text{predicted\_cer} &= \sum_{j=16}^{20} \text{predicted\_cer}_j

\end{align*}

$$


**Step 5: Compute FLR from extrapolated CER by considering interleaving factor**

$$

\text{FEC\_FLR\_PREDICTED} =

\begin{cases}

1.125 \times \text{predicted\_cer} & \text{if } X=1 \\

2.125 \times \text{predicted\_cer} & \text{if } X=2

\end{cases}

$$


**Step 6: Store FEC_FLR_PREDICTED in the COUNTER_DB:RATES table**