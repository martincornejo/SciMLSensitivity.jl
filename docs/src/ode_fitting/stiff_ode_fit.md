# Parameter Estimation on Highly Stiff Systems

This tutorial goes into training a model on stiff chemical reaction system data.

## Copy-Pasteable Code

Before getting to the explanation, here's some code to start with. We will
follow a full explanation of the definition and training process:

```@example
using DifferentialEquations, DiffEqFlux, Optimization, OptimizationOptimJL, LinearAlgebra
using ForwardDiff
using DiffEqBase: UJacobianWrapper
using Plots
function rober(du,u,p,t)
    y₁,y₂,y₃ = u
    k₁,k₂,k₃ = p
    du[1] = -k₁*y₁+k₃*y₂*y₃
    du[2] =  k₁*y₁-k₂*y₂^2-k₃*y₂*y₃
    du[3] =  k₂*y₂^2
    nothing
end

p = [0.04,3e7,1e4]
u0 = [1.0,0.0,0.0]
prob = ODEProblem(rober,u0,(0.0,1e5),p)
sol = solve(prob,Rosenbrock23())
ts = sol.t
Js = map(u->I + 0.1*ForwardDiff.jacobian(UJacobianWrapper(rober, 0.0, p), u), sol.u)

function predict_adjoint(p)
    p = exp.(p)
    _prob = remake(prob,p=p)
    Array(solve(_prob,Rosenbrock23(autodiff=false),saveat=ts,sensealg=QuadratureAdjoint(autojacvec=ReverseDiffVJP(true))))
end

function loss_adjoint(p)
    prediction = predict_adjoint(p)
    prediction = [prediction[:, i] for i in axes(prediction, 2)]
    diff = map((J,u,data) -> J * (abs2.(u .- data)) , Js, prediction, sol.u)
    loss = sum(abs, sum(diff)) |> sqrt
    loss, prediction
end

callback = function (p,l,pred) #callback function to observe training
    println("Loss: $l")
    println("Parameters: $(exp.(p))")
    # using `remake` to re-create our `prob` with current parameters `p`
    plot(solve(remake(prob, p=exp.(p)), Rosenbrock23())) |> display
    return false # Tell it to not halt the optimization. If return true, then optimization stops
end

initp = ones(3)
# Display the ODE with the initial parameter values.
callback(initp,loss_adjoint(initp)...)

adtype = Optimization.AutoZygote()
optf = Optimization.OptimizationFunction((x,p)->loss_adjoint(x), adtype)
optprob = Optimization.OptimizationProblem(optf, initp)

res = Optimization.solve(optprob, ADAM(0.01), callback = callback, maxiters = 300)

optprob2 = Optimization.OptimizationProblem(optf, res.u)

res2 = Optimization.solve(optprob2, BFGS(), callback = callback, maxiters = 30, allow_f_increases=true)
println("Ground truth: $(p)\nFinal parameters: $(round.(exp.(res2.u), sigdigits=5))\nError: $(round(norm(exp.(res2.u) - p) ./ norm(p) .* 100, sigdigits=3))%")
```

Output:
```
Ground truth: [0.04, 3.0e7, 10000.0]
Final parameters: [0.040002, 3.0507e7, 10084.0]
Error: 1.69%
```

## Explanation

First, let's get a time series array from the Robertson's equation as data.

```@example stifffit
using DifferentialEquations, DiffEqFlux, Optimization, OptimizationOptimJL, LinearAlgebra
using ForwardDiff
using DiffEqBase: UJacobianWrapper
using Plots
function rober(du,u,p,t)
    y₁,y₂,y₃ = u
    k₁,k₂,k₃ = p
    du[1] = -k₁*y₁+k₃*y₂*y₃
    du[2] =  k₁*y₁-k₂*y₂^2-k₃*y₂*y₃
    du[3] =  k₂*y₂^2
    nothing
end

p = [0.04,3e7,1e4]
u0 = [1.0,0.0,0.0]
prob = ODEProblem(rober,u0,(0.0,1e5),p)
sol = solve(prob,Rosenbrock23())
ts = sol.t
Js = map(u->I + 0.1*ForwardDiff.jacobian(UJacobianWrapper(rober, 0.0, p), u), sol.u)
```

Note that we also computed a shifted and scaled Jacobian along with the
solution. We will use this matrix to scale the loss later.

We fit the parameters in log space, so we need to compute `exp.(p)` to get back
the original parameters.

```@example stifffit
function predict_adjoint(p)
    p = exp.(p)
    _prob = remake(prob,p=p)
    Array(solve(_prob,Rosenbrock23(autodiff=false),saveat=ts,sensealg=QuadratureAdjoint(autojacvec=ReverseDiffVJP(true))))
end

function loss_adjoint(p)
    prediction = predict_adjoint(p)
    prediction = [prediction[:, i] for i in axes(prediction, 2)]
    diff = map((J,u,data) -> J * (abs2.(u .- data)) , Js, prediction, sol.u)
    loss = sum(abs, sum(diff)) |> sqrt
    loss, prediction
end
```

The difference between the data and the prediction is weighted by the transformed
Jacobian to do a relative scaling of the loss.

We define a callback function.
```@example stifffit
callback = function (p,l,pred) #callback function to observe training
    println("Loss: $l")
    println("Parameters: $(exp.(p))")
    # using `remake` to re-create our `prob` with current parameters `p`
    plot(solve(remake(prob, p=exp.(p)), Rosenbrock23())) |> display
    return false # Tell it to not halt the optimization. If return true, then optimization stops
end
```

We then use a combination of `ADAM` and `BFGS` to minimize the loss function to
accelerate the optimization. The initial guess of the parameters are chosen to
be `[1, 1, 1.0]`.
```@example stifffit
initp = ones(3)
# Display the ODE with the initial parameter values.
callback(initp,loss_adjoint(initp)...)

adtype = Optimization.AutoZygote()
optf = Optimization.OptimizationFunction((x,p)->loss_adjoint(x), adtype)

optprob = Optimization.OptimizationProblem(optf, initp)
res = Optimization.solve(optprob, ADAM(0.01), callback = callback, maxiters = 300)

optprob2 = Optimization.OptimizationProblem(optf, res.u)
res2 = Optimization.solve(optprob2, BFGS(), callback = callback, maxiters = 30, allow_f_increases=true)
```

Finally, we can analyze the difference between the fitted parameters and the
ground truth.
```@example stifffit
println("Ground truth: $(p)\nFinal parameters: $(round.(exp.(res2.u), sigdigits=5))\nError: $(round(norm(exp.(res2.u) - p) ./ norm(p) .* 100, sigdigits=3))%")
```

It gives the output
```
Ground truth: [0.04, 3.0e7, 10000.0]
Final parameters: [0.040002, 3.0507e7, 10084.0]
Error: 1.69%
```