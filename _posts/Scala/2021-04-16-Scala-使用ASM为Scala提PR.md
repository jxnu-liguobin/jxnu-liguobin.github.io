---
title: 使用ASM为Scala提PR
categories:
  - Scala
tags: [Scala，编译器，字节码]
description: 实战ASM，为Scala编译器提PR
---

动态代理只知道cglib和jdk代理就可以？说明还不够卷。


1.背景

伴生对象（无同名类时，则为单例对象）用于实现Java静态方法，会为其生成两个class文件，文件名带\$后缀的是真正的实现，包含实例方法，文件名不带\$的是代理类，包含静态方法。

需求：将伴生对象的代理类的方法参数名写入字节码中。

相关代码在Scala语言编译器模块（compiler），backend包中的`BCodeHelpers.scala`文件。这里的asm是Scala的，而不是原来的asm，原理相同。

首先需要了解asm基本的操作，asm分为事件API和Tree API，前者使用访问者模式将数据结构和操作分离，这里我们使用访问者API实现会更加方便，同时源码该处代码也使用了访问者API。

2.通过运行并debug编译器（compiler），找到代理类的字节码操作在哪块地方。

在`BCodeSkelBuilder.scala`中可以找到初始化类的函数

```scala
    /*
     * must-single-thread
     */
    private def initJClass(jclass: asm.ClassVisitor): Unit = {

      val bType = classBTypeFromSymbol(claszSymbol)
      val superClass = bType.info.get.superClass.getOrElse(ObjectRef).internalName
      val interfaceNames = bType.info.get.interfaces.map(_.internalName)

      val flags = javaFlags(claszSymbol)

      val thisSignature = getGenericSignature(claszSymbol, claszSymbol.owner)
      cnode.visit(backendUtils.classfileVersion.get, flags,
                  thisBType.internalName, thisSignature,
                  superClass, interfaceNames.toArray)

      if (emitSource) {
        cnode.visitSource(cunit.source.toString, null /* SourceDebugExtension */)
      }

      enclosingMethodAttribute(claszSymbol, internalName, methodBTypeFromSymbol(_).descriptor) match {
        case Some(EnclosingMethodEntry(className, methodName, methodDescriptor)) =>
          cnode.visitOuterClass(className, methodName, methodDescriptor)
        case _ => ()
      }

      val ssa = getAnnotPickle(thisBType.internalName, claszSymbol)
      cnode.visitAttribute(if (ssa.isDefined) pickleMarkerLocal else pickleMarkerForeign)
      emitAnnotations(cnode, claszSymbol.annotations ++ ssa)

      if (isCZStaticModule || isCZParcelable) {

        if (isCZStaticModule) { addModuleInstanceField() }

      } else {

        if (!settings.noForwarders) {
          val lmoc = claszSymbol.companionModule
          // add static forwarders if there are no name conflicts; see bugs #363 and #1735
          if (lmoc != NoSymbol) {
            // it must be a top level class (name contains no $s)
            val isCandidateForForwarders = {
              exitingPickler { !(lmoc.name.toString contains '$') && lmoc.hasModuleFlag && !lmoc.isNestedClass }
            }
            if (isCandidateForForwarders) {
              // 这里是入口，根据注释也能知道，addForwarders就是添加转发的地方。转发就是上面说的代理类。
              log(s"Adding static forwarders from '$claszSymbol' to implementations in '$lmoc'")
              addForwarders(cnode, thisBType.internalName, lmoc.moduleClass)
            }
          }
        }

      }

      // the invoker is responsible for adding a class-static constructor.

    } // end of method initJClass
```
继续看`addForwarders`方法，这个方法的注释很详细，说明了添加转发的条件。
```scala
    /* Add forwarders for all methods defined in `module` that don't conflict
     *  with methods in the companion class of `module`. A conflict arises when
     *  a method with the same name is defined both in a class and its companion object:
     *  method signature is not taken into account.
     *
     * must-single-thread
     */
    def addForwarders(jclass: asm.ClassVisitor, jclassName: String, moduleClass: Symbol): Unit = {
      assert(moduleClass.isModuleClass, moduleClass)

      val linkedClass = moduleClass.companionClass
      lazy val conflictingNames: Set[Name] = {
        (linkedClass.info.members collect { case sym if sym.name.isTermName => sym.name }).toSet
      }

      // Before erasure * to exclude bridge methods. Excluding them by flag doesn't work, because then
      // the method from the base class that the bridge overrides is included (scala/bug#10812).
      // * Using `exitingUncurry` (not `enteringErasure`) because erasure enters bridges in traversal,
      //   not in the InfoTransform, so it actually modifies the type from the previous phase.
      //   Uncurry adds java varargs, which need to be included in the mirror class.
      val members = exitingUncurry(moduleClass.info.membersBasedOnFlags(BCodeHelpers.ExcludedForwarderFlags, symtab.Flags.METHOD))
      for (m <- members) {
        val excl = m.isDeferred || m.isConstructor || m.hasAccessBoundary ||
          { val o = m.owner; (o eq ObjectClass) || (o eq AnyRefClass) || (o eq AnyClass) } ||
          conflictingNames(m.name)
        // 当可以添加转发器时才添加
        if (!excl) addForwarder(jclass, moduleClass, m)
      }
    }
```
找到了在哪添加转发方法，使用我们的asm搞就行。代码比较复杂，需要清楚了解上下文，所以还是贴完整代码吧。
```scala
    /* Add a forwarder for method m. Used only from addForwarders().
     * 这个注释很明确了，为方法m生成转发方法，该方法仅为addForwarders使用。
     * must-single-thread
     */
    private def addForwarder(jclass: asm.ClassVisitor, moduleClass: Symbol, m: Symbol): Unit = {
      //静态转发方法的的泛型签名  
      def staticForwarderGenericSignature: String = {
        // scala/bug#3452 Static forwarder generation uses the same erased signature as the method if forwards to.
        // By rights, it should use the signature as-seen-from the module class, and add suitable
        // primitive and value-class boxing/unboxing.
        // But for now, just like we did in mixin, we just avoid writing a wrong generic signature
        // (one that doesn't erase to the actual signature). See run/t3452b for a test case.
        val memberTpe = enteringErasure(moduleClass.thisType.memberInfo(m))
        val erasedMemberType = erasure.erasure(m)(memberTpe)
        if (erasedMemberType =:= m.info)
          getGenericSignature(m, moduleClass, memberTpe)
        else null
      }
      
      val moduleName     = internalName(moduleClass)
      val methodInfo     = moduleClass.thisType.memberInfo(m)
      val paramTypes     = methodInfo.paramTypes
      val paramJavaTypes = BType.newArray(paramTypes.length)
      mapToArray(paramTypes, paramJavaTypes, 0)(typeToBType)
      // val paramNames     = 0 until paramJavaTypes.length map ("x_" + _)

      /* Forwarders must not be marked final,
       *  as the JVM will not allow redefinition of a final static method,
       *  and we don't know what classes might be subclassing the companion class.  See scala/bug#4827.
       */
      // TODO: evaluate the other flags we might be dropping on the floor here.
      // TODO: ACC_SYNTHETIC ?
      val flags = GenBCode.PublicStatic |
        (if (m.isVarargsMethod) asm.Opcodes.ACC_VARARGS else 0) |
        (if (m.isDeprecated) asm.Opcodes.ACC_DEPRECATED else 0)

      // TODO needed? for(ann <- m.annotations) { ann.symbol.initialize }
      val jgensig = staticForwarderGenericSignature

      val (throws, others) = partitionConserve(m.annotations)(_.symbol == definitions.ThrowsClass)
      val thrownExceptions: List[String] = getExceptions(throws)

      val jReturnType = typeToBType(methodInfo.resultType)
      val mdesc = MethodBType(paramJavaTypes, jReturnType).descriptor
      val mirrorMethodName = m.javaSimpleName.toString
      // jclass是ClassVisitor，用asm做过example的应该都知道，访问类的方法当然需要先得到类的访问器。
      val mirrorMethod: asm.MethodVisitor = jclass.visitMethod(
        flags,
        mirrorMethodName,
        mdesc,
        jgensig,
        mkArray(thrownExceptions)
      )
      // 添加的代码 - 开始
      // 表示本地变量的参数的可见域从这里开始，因为转发方法是静态的，且没有任何其他本地变量，所以我们只需要将静态方法的参数写入本地变量表即可。
      // 方法的变量的作用域就是方法开始到方法结束，这也很好理解，所以最后还有一个end
      val localScopeStart: Label = new Label()

      // 根据作用域将当前方法的所有参数写入变量表
      def emitMethodParameters(localScopeStart: Label, localScopeEnd: Label): Unit = {
        var i = 0
        var next = 0
        // 使用方法的信息中的参数来遍历，因为很显然，我们需要写入每个方法的每个参数。
        m.info.params.foreach { p =>
        // 重点，参数名，参数类型，签名，参数起始可见域，最终可见域，参数的solt。
        // 这里的签名可以为空，有参数类型即可，最关键的是solt，它与参数类型的大小有关还可以复用。可以看到，这里使用了paramJavaTypes(i).size。
        // 在Scala中，其实paramJavaTypes(i).size只有三种可能， UNIT => 0，LONG | DOUBLE => 2，_ => 1
          mirrorMethod.visitLocalVariable(p.name.encodedName.toString, paramJavaTypes(i).descriptor, null, localScopeStart, localScopeEnd, next)
          next += paramJavaTypes(i).size
          i += 1
        }
      }
      // 添加的代码 - 结束

      emitParamNames(mirrorMethod, m.info.params)
      emitAnnotations(mirrorMethod, others)
      emitParamAnnotations(mirrorMethod, m.info.params.map(_.annotations))

      mirrorMethod.visitCode()

      mirrorMethod.visitFieldInsn(asm.Opcodes.GETSTATIC, moduleName, strMODULE_INSTANCE_FIELD, classBTypeFromSymbol(moduleClass).descriptor)

      var index = 0
      for(jparamType <- paramJavaTypes) {
        mirrorMethod.visitVarInsn(jparamType.typedOpcode(asm.Opcodes.ILOAD), index)
        assert(!jparamType.isInstanceOf[MethodBType], jparamType)
        index += jparamType.size
      }

      mirrorMethod.visitMethodInsn(asm.Opcodes.INVOKEVIRTUAL, moduleName, mirrorMethodName, methodBTypeFromSymbol(m).descriptor, false)
      mirrorMethod.visitInsn(jReturnType.typedOpcode(asm.Opcodes.IRETURN))

      mirrorMethod.visitMaxs(0, 0) // just to follow protocol, dummy arguments

      // 添加的代码 - 开始
      // 方法访问结束，标记最终可见域，同时执行我们的emitMethodParameters方法。
      val localScopeEnd = new Label()
      mirrorMethod.visitLabel(localScopeEnd)

      emitMethodParameters(localScopeStart, localScopeEnd)
      // 添加的代码 - 结束
      
      mirrorMethod.visitEnd()

    }
```

其实不必了解所有细节，该需求相对比较独立，只要保证本地变量表保存的没问题即可。

3.最终编译器负责人将该代码优化了一下并合进了。

主要去掉了我单独写的`emitMethodParameters`方法，同时使得代码更加函数式。

```scala
// 使用了tap辅助方法，使得调用起来更顺手，不用定义单独的label
// 可能是考虑到该方法不太可能存在重用，所以直接把代码写到最后了。
val codeStart: Label = new Label().tap(mirrorMethod.visitLabel)
  mirrorMethod.visitFieldInsn(asm.Opcodes.GETSTATIC, moduleName, strMODULE_INSTANCE_FIELD, classBTypeFromSymbol(moduleClass).descriptor)

  var index = 0
  for(jparamType <- paramJavaTypes) {
    mirrorMethod.visitVarInsn(jparamType.typedOpcode(asm.Opcodes.ILOAD), index)
    assert(!jparamType.isInstanceOf[MethodBType], jparamType)
    index += jparamType.size
  }

  mirrorMethod.visitMethodInsn(asm.Opcodes.INVOKEVIRTUAL, moduleName, mirrorMethodName, methodBTypeFromSymbol(m).descriptor, false)
  mirrorMethod.visitInsn(jReturnType.typedOpcode(asm.Opcodes.IRETURN))
  val codeEnd = new Label().tap(mirrorMethod.visitLabel)

  // 使用lazyZip和foldLeft，移除了 for循环和var i，更加符合函数式。
  methodInfo.params.lazyZip(paramJavaTypes).foldLeft(0) {
    case (idx, (p, tp)) =>
      mirrorMethod.visitLocalVariable(p.name.encoded, tp.descriptor, null, codeStart, codeEnd, idx)
      idx + tp.size
  }
```
