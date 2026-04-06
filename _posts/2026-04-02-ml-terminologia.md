---
title: Visão geral do aprendizado de máquina
description: Notes and lessons learned while setting up a Linux Kernel development environment using QEMU and libvirt.
date: 2030-02-25 15:25:00 +/-TTTT
categories: [Machine learning]
tags: [Andrew Ng, Regression, Classification]
author: <author_id> # for single entry
---

## Terminologies

    Training Set: This is the collection of data you use to teach your model. Think of it like a set of practice problems with known answers. For example, a list of houses with their sizes and prices.

    Input (x): This is the feature or characteristic you use to make a prediction. In the house price example, the input is the size of the house (in square feet).

    Output (y): This is the value you want to predict. For the house example, it’s the price of the house (in thousands of dollars).

    Training Example ($x^(i), y^(i)$): Each row in your training set is a training example. The superscript (i) just means the i-th example, like the first, second, or third house in the list.

    m: This is the total number of training examples in your dataset. For instance, if you have 47 houses, then m = 47.

    y-hat (ŷ): predicted value; valor estimado ou previsão feita pelo modelo (valor previsto).

    w (Weight or Coefficient): Representa o peso ou coeficiente da variável de entrada x. Ele determina a inclinação (slope) da linha no gráfico. Um valor maior de w significa que a linha é mais inclinada.

    b (Bias or Intercept): É o termo constante que desloca a linha para cima ou para baixo no gráfico. Também é chamado de intercepto y, pois é o ponto onde a linha cruza o eixo y quando x é zero.