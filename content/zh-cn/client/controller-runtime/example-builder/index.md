---
title: "使用builder的基础controller"
linkTitle: "builder例子"
weight: 30
date: 2021-02-01
description: >
  
---



> https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/builder#example-Builder



## 概述

Package builder包装了其他的控制器运行时库，并展示了构建普通控制器的简单模式。

如果项目在未来需要更多的自定义行为，使用builder包构建的项目可以在底层包的基础上进行简单的重构。

## 例子

这个例子创建了一个简单的应用程序 ControllerManagedBy，为ReplicaSets 和 Pod 进行了配置。

* 为ReplicaSets创建一个新的应用程序，管理 ReplicaSet 所拥有的 Pod，并调用到 ReplicaSetReconciler。

* 启动应用程序。



```go
package main

import (
	"context"
	"fmt"
	"os"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"

	"sigs.k8s.io/controller-runtime/pkg/builder"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/client/config"

	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"
	"sigs.k8s.io/controller-runtime/pkg/manager"
	"sigs.k8s.io/controller-runtime/pkg/manager/signals"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
)

func main() {
	logf.SetLogger(zap.New())

	var log = logf.Log.WithName("builder-examples")

	mgr, err := manager.New(config.GetConfigOrDie(), manager.Options{})
	if err != nil {
		log.Error(err, "could not create manager")
		os.Exit(1)
	}

	err = builder.
		ControllerManagedBy(mgr).  // Create the ControllerManagedBy
		For(&appsv1.ReplicaSet{}). // ReplicaSet is the Application API
		Owns(&corev1.Pod{}).       // ReplicaSet owns Pods created by it
		Complete(&ReplicaSetReconciler{
			Client: mgr.GetClient(),
		})
	if err != nil {
		log.Error(err, "could not create controller")
		os.Exit(1)
	}

	if err := mgr.Start(signals.SetupSignalHandler()); err != nil {
		log.Error(err, "could not start manager")
		os.Exit(1)
	}
}

// ReplicaSetReconciler is a simple ControllerManagedBy example implementation.
type ReplicaSetReconciler struct {
	client.Client
}

// Implement the business logic:
// This function will be called when there is a change to a ReplicaSet or a Pod with an OwnerReference
// to a ReplicaSet.
//
// * Read the ReplicaSet
// * Read the Pods
// * Set a Label on the ReplicaSet with the Pod count.
func (a *ReplicaSetReconciler) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {

	rs := &appsv1.ReplicaSet{}
	err := a.Get(ctx, req.NamespacedName, rs)
	if err != nil {
		return reconcile.Result{}, err
	}

	pods := &corev1.PodList{}
	err = a.List(ctx, pods, client.InNamespace(req.Namespace), client.MatchingLabels(rs.Spec.Template.Labels))
	if err != nil {
		return reconcile.Result{}, err
	}

	rs.Labels["pod-count"] = fmt.Sprintf("%v", len(pods.Items))
	err = a.Update(ctx, rs)
	if err != nil {
		return reconcile.Result{}, err
	}

	return reconcile.Result{}, nil
}
```



仅有 

```go
package main

import (
	"context"
	"fmt"
	"os"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

	"sigs.k8s.io/controller-runtime/pkg/builder"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/client/config"

	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"
	"sigs.k8s.io/controller-runtime/pkg/manager"
	"sigs.k8s.io/controller-runtime/pkg/manager/signals"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
)

func main() {
	logf.SetLogger(zap.New())

	var log = logf.Log.WithName("builder-examples")

	mgr, err := manager.New(config.GetConfigOrDie(), manager.Options{})
	if err != nil {
		log.Error(err, "could not create manager")
		os.Exit(1)
	}

	cl := mgr.GetClient()
	err = builder.
		ControllerManagedBy(mgr).                  // Create the ControllerManagedBy
		For(&appsv1.ReplicaSet{}).                 // ReplicaSet is the Application API
		Owns(&corev1.Pod{}, builder.OnlyMetadata). // ReplicaSet owns Pods created by it, and caches them as metadata only
		Complete(reconcile.Func(func(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
			// Read the ReplicaSet
			rs := &appsv1.ReplicaSet{}
			err := cl.Get(ctx, req.NamespacedName, rs)
			if err != nil {
				return reconcile.Result{}, client.IgnoreNotFound(err)
			}

			// List the Pods matching the PodTemplate Labels, but only their metadata
			var podsMeta metav1.PartialObjectMetadataList
			err = cl.List(ctx, &podsMeta, client.InNamespace(req.Namespace), client.MatchingLabels(rs.Spec.Template.Labels))
			if err != nil {
				return reconcile.Result{}, client.IgnoreNotFound(err)
			}

			// Update the ReplicaSet
			rs.Labels["pod-count"] = fmt.Sprintf("%v", len(podsMeta.Items))
			err = cl.Update(ctx, rs)
			if err != nil {
				return reconcile.Result{}, err
			}

			return reconcile.Result{}, nil
		}))
	if err != nil {
		log.Error(err, "could not create controller")
		os.Exit(1)
	}

	if err := mgr.Start(signals.SetupSignalHandler()); err != nil {
		log.Error(err, "could not start manager")
		os.Exit(1)
	}
}
```

