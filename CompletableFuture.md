## 一、任务顺序执行

1. `thenRun(Runnable action)`：任务 A 执行完执行 B，并且 B 不依赖 A 的结果。
2. `thenAccept(Consumer<? super T> action)`：任务 A 执行完执行 B，B 依赖 A 的结果，但是任务 B 没有返回值。
3. `thenApply(Function<? super T,? extends U>fn)`：任务 A 执行完执行 B，B 依赖 A 的结果，同时任务 B 有返回值。

## 二、异常处理

1. `exceptionally(Function<Throwable, ? extends T> fn)`：当任务 A 抛出异常时，通过`exceptionally`方法处理异常并将返回的新结果传递给任务 B。

    ```java
    CompletableFuture<String> completableFuture = CompletableFuture.supplyAsync(() -> {
                        throw new RuntimeException();
                    })
                    .exceptionally(ex -> "errorResultA")
                    .thenApply(resultA -> resultA + " resultB")
                    .thenApply(resultB -> resultB + " resultC")
                    .thenApply(resultC -> resultC + " resultD");

            completableFuture.join();
    ```

2. `handle(BiFunction<? super T, Throwable, ? extends U> fn)`：`re`和`throwable`必然有一个是`null`，它们分别代表正常的执行结果和异常的情况。

    ```java
    CompletableFuture<String> completableFuture = CompletableFuture.supplyAsync(() -> "resultA")
                    .thenApply(resultA -> resultA + " resultB")
                    .thenApply(resultB -> {
                        throw new RuntimeException();
                    })
                    .handle((re, throwable) -> {
                        if (throwable != null) {
                            return "errorResultC";
                        }
                        return re;
                    })
                    .thenApply(resultC -> resultC + " resultD");

            completableFuture.join();
    ```

## 三、获取任务执行结果

1. `thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action)`：表示后续的处理不需要返回值。
2. `thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)`：表示需要返回值。

    ```java
    CompletableFuture<String> completableFutureA = CompletableFuture.supplyAsync(() -> {
                System.out.println("processing a...");
                return "Hello";
            });

            CompletableFuture<String> completableFutureB = CompletableFuture.supplyAsync(() -> {
                System.out.println("processing b...");
                return " World";
            });

            CompletableFuture<String> completableFutureC = CompletableFuture.supplyAsync(() -> {
                System.out.println("processing c...");
                return ", I'm robot!";
            });
            
            completableFutureA.thenCombine(completableFutureB, (resultA, resultB) -> {
                // Hello World
                System.out.println(resultA + resultB);
                return resultA + resultB;
            }).thenCombine(completableFutureC, (resultAB, resultC) -> {
                // Hello World, I'm robot!
                System.out.println(resultAB + resultC);
                return resultAB + resultC;
            });
    ```

3. `thenCompose(Function<? super T, ? extends CompletionStage<U>> fn)`：`thenCombine`是把**没有依赖关系**的执行结果进行简单的**聚合**，但是`thenCompose`更像是把多个已有的`CompletableFuture`实例组合成一个**处理链**，将前一个任务的执行结果作为下一个任务的参数，它们之间存在着**先后顺序**。

    ```java
    CompletableFuture<String> result = CompletableFuture.supplyAsync(() -> "Hello")
                    .thenCompose(resultA -> CompletableFuture.supplyAsync(() -> resultA + " World"))
                    .thenCompose(resultAB -> CompletableFuture.supplyAsync(() -> resultAB + ", I'm robot!"));

            // Hello World, I'm robot!
            System.out.println(result.join());
    ```
