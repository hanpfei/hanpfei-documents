---
title: SOF 拓扑 2.0
date: 2024-02-25 19:08:29
categories: Linux 内核
tags:
- Linux 内核
---

这是现有 ALSA conf 拓扑格式之上的高级关键字扩展，旨在：

 * 通过提供高级的 “classes” 来简化 ALSA conf 拓扑定义。通过这种方式，拓扑设计者可以为经常定义的对象编写更少的配置。
 * 允许简单地重用对象。定义一次并重用（如 M4），能够更改默认的对象配置属性。
 * 允许数据类型和值验证。现在还没有这样做，并且经常出现在固件错误报告中。

## 要素

典型的 2.0 配置文件由以下组件组成：

 * 类（Classes）
 * 对象（Objects）
 * 参数（Arguments）
 * 条件包含（Conditional includes）

### 类

今天的拓扑有一些常见的定义，常常通过对配置进行细微的改变来重用，例如 widgets（组件）、流水线、dais、pcm 和 controls。拓扑 2.0 引入了可重用的类定义的概念，你可以使用它来创建常用的拓扑对象。类通过一个新的关键字 `Class` 来定义。

类定义始终以 `Class` 关键字开头，后跟两个节点。第一个节点包含类的组，第二个节点包含类名。如：
```
Class.Base.data {}
```

请注意，‘.’ 是 alsaconf 语法中的节点分隔符。在上面的行中，`Base` 是类的组，`data` 是类名。目前，alsatplg 编译器支持以下类组：widget、pipeline、DAI、control 和 base。大多数常用的拓扑对象都可以分为这些组之一。如果需要新的类组，则应更新 alsatplg 编译器以添加对其的支持。

#### 类要素

简约的类定义应包含以下内容：

 * 使用关键字 `DefineAttribute` 声明的一个或多个属性。属性是用于描述对象的参数。比如：
```
DefineAttribute."name" {
        type "string"
}
```

“name” 是字符串类型的属性。

 * 带有构造函数数组和唯一属性名称的基本属性限定符。属性限定符应该在类定义的 `attributes {}` 节点中声明。
```
# attribute qualifiers
attributes {
        #
        # This tells the compiler how to construct the object's name. For example, if the
        # name attribute is set to "EQIIR-Coefficients", the object name will be
        # constructed as "class_name.EQIIR-Coefficients"
        #
        !constructor [
                "name"
        ]
        #
        # objects of the same class instantiated within the same alsaconf node have unique
        # name attribute
        #
        unique  "name"
}
```

#### 一个简单的类

以下示例演示了具有两个属性和限定符的简单类定义：
```
Class.Base."data" {

        # name for the data object
        DefineAttribute."name" {
                type    "string"
        }

        # bytes data
        DefineAttribute."bytes" {
                type    "string"
        }

        # attribute qualifiers
        attributes {
                #
                # This tells the compiler how to construct the object's name. For example, if the
                # name attribute is set to "EQIIR-Coefficients", the object name will be
                # constructed as "data.EQIIR-Coefficients"
                #
                !constructor [
                        "name"
                ]
                #
                # data objects instantiated within the same alsaconf node should have unique
                # name attribute
                #
                unique  "name"
        }
}
```

`data` 类定义属于 `base` 类组，它包含两个属性，name 和 bytes，两者的类型都是 `string`。默认情况下，除非另有指定，否则所有属性都是 `integer` 类型的，如上例所示。目前，拓扑 2.0 仅支持 `integer` 和 `string` 类型的属性。

属性限定符用于描述如何从类定义实例化对象并验证属性值。

在上面的定义中，`constructor` 数组告诉编译器如何构建对象的名称。使用名称 `EQIIR-Coefficients` 实例化的数据对象将被赋予名称 `data.EQIIR-Coefficients`，即类名后跟 ‘.’，后跟由 ‘.’ 分隔的构造函数属性值。

`unique` 限定符指示在同一个 alsaconf 节点中实例化的多个数据对象的 `name` 属性应该具有唯一的值。如果在同一个 alsaconf 节点内实例化具有相同 name 属性的两个数据对象，则不会发生错误，但两个对象实例将被合并。此外，第二个实例中的属性值将覆盖第一个实例中的属性值。因此，拓扑编写者有责任确保同一父节点中的多个实例具有不同的唯一属性值。

让我们考虑另一个类定义示例，属于类组 `Widget` 的 `pga` widget：
```
Class.Widget."pga" {
        #
        # Pipeline ID for the pga widget object
        #
        DefineAttribute."index" {}

        #
        # pga object instance
        #
        DefineAttribute."instance" {}

        # attribute qualifiers
        attributes {
                #
                # The PGA widget name is constructed using the index and instance
                # attributes. For ex: "pga.1.1" or "pga.10.2" etc.
                #
                !constructor [
                        "index"
                        "instance"
                ]

                #
                # pga widget objects instantiated within the same alsaconf node should have unique
                # instance attribute
                #
                unique  "instance"
        }
}
```

请注意，pga 对象名称是由类名 pga 后跟两个属性值，index 和 instance 构成的。比如，`pga.1.1`。由于定义未指定类型，默认情况下，这两个属性的类型都为 `integer`。实际上，唯一的 instance 属性也应该是构造函数的一部分。

#### 属性默认值

可选地，可以扩展类定义以为其属性提供默认值。让我们为 `pga` 类添加一个类型为 `string` 的属性 `uuid`，并给它一个默认值：
```
Class.Widget."pga" {
        #
        # Pipeline ID for the pga widget object
        #
        DefineAttribute."index" {}

        #
        # pga object instance
        #
        DefineAttribute."instance" {}

        DefineAttribute."uuid" {
                type "string"
        }

        # attribute qualifiers
        attributes {
                #
                # The PGA widget name is constructed using the index and instance
                # attributes. For ex: "pga.1.1" or "pga.10.2" etc.
                #
                !constructor [
                        "index"
                        "instance"
                ]

                #
                # pga widget objects instantiated within the same alsaconf node should have unique
                # instance attribute
                #
                unique  "instance"
        }

        # default attribute values
        uuid                    "7e:67:7e:b7:f4:5f:88:41:af:14:fb:a8:bd:bf:86:82"

}
```

所有 pga 对象将自动获得默认的 uuid，如上面在类定义中指定的那样。

#### 高级属性限定符

除了强制性的基本属性限定符外，你还可以使用以下的高级关键字限定类定义中的属性：

 * **Mandatory**：应在对象实例中为 mandatory 限定的属性提供一个值，否则 alsatplg 编译器将发出错误。类定义中具有默认值的对象不需要被限定为 mandatory。另请注意，构造函数数组中的属性默认为强制的，因为它们是构建对象名称所必需的。
 * **Immutable**：在类定义中设置的属性值，不能在对象实例中修改。
 * **Deprecated**：已弃用且不应在对象实例中设置的属性。
 * **Automatic**：其值由 alsatplg 编译器计算的属性。

让我们在 pga 类定义中添加一些额外的属性和高级限定符：
```
Class.Widget."pga" {
        # attribute definitions
        DefineAttribute.instance {
                type    "integer"
        }
        DefineAttribute.index {
                type    "integer"
        }
        DefineAttribute."type" {
                type    "string"
        }
        DefineAttribute."uuid" {
                type    "string"
        }
        DefineAttribute."preload_count" {}

        # attribute qualifiers
        attributes {
                #
                # The PGA widget name is constructed using the index and instance attributes.
                # For ex: "pga.1.1" or "pga.10.2" etc.
                #
                !constructor [
                        "index"
                        "instance"
                ]

                #
                # immutable attributes should be given default values and cannot be modified in the object instance
                #
                !immutable [
                        "uuid"
                        "type"
                ]

                #
                # deprecated attributes should not be added in the object instance
                #
                !deprecated [
                        "preload_count"
                ]

                #
                # pga widget objects instantiated within the same alsaconf node should have
                # unique instance attribute
                #
                unique  "instance"
        }

        # default attribute values
        type            "pga"
        uuid            "7e:67:7e:b7:f4:5f:88:41:af:14:fb:a8:bd:bf:86:82"
}
```

#### 自动属性

在某些情况下，属性值取决于其它属性值，并且需要在构建时计算。在类定义中这种属性用 `automatic` 关键字修饰。请参阅 [buffer](https://github.com/thesofproject/sof/blob/main/tools/topology/topology2/include/components/buffer.conf) 以获取完整的类定义。
```
Class.Widget."buffer" {
        # Other attributes skipped for simplicity.

        #
        # Buffer size in bytes. Will be calculated based on the parameters of the pipeline to in which the
        # buffer object belongs
        #
        DefineAttribute."size" {
                # Token reference and type
                token_ref       "sof_tkn_buffer.word"
        }

        attributes {
                #
                # size attribute value for buffer objects is computed in the compiler
                #
                !automatic [
                        "size"
                ]
        }
}
```

在上面的示例中，`buffer` 的 `size` 属性值基于 buffer 所属的流水线的参数计算。目前，alsatplg 编译器仅支持计算 buffer 对象的自动属性 `size`。如有必要，应在 alsatplg 编译器中添加对新类定义中的自动属性的支持。

#### 属性约束

拓扑 2.0 的重要特性之一是验证提供给对象的值。这通过给属性定义添加约束来实现。可以使用 `constraints` 关键字来给属性添加约束：
```
DefineAttribute."foo" {
        constraints {}
}
```

目前，支持三种类型的约束：

 * **min**：属性的最小值，仅适用于整数类型属性。
 * **max**：属性的最大值，仅适用于整数类型属性。
比如，pga 类定义可以扩展为具有属性 `ramp_step_ms`，其 min 和 max 值如下：
```
DefineAttribute."ramp_step_ms" {
        constraints {
                min 200
                max 500
        }
}
```
 * **valid values**：可接受的人类可读值的数组，仅适用于字符串类型的属性。
例如，pga 类可以具有带有预定义值的 `ramp_step_type` 属性，如下所示：
```
DefineAttribute."ramp_step_type" {
        type    "string"
        constraints {
                !valid_values [
                        "linear"
                        "log"
                        "linear_zc"
                        "log_zc"
                ]
        }
}
```

当使用不属于 `valid_values` 数组的 `ramp_step_type` 值实例化 pga 类时，alsatplg 编译器会发出错误以及有效值列表。

#### 具有 token 引用的属性

通常，许多对象包含由元组数组集合组成的私有数据部分。类定义中的某些属性可能需要打包到元组数组中。此类属性通过 `token_ref` 节点进行标识，该节点包含属性应构建到的元组数组的名称。例如，pga 类中的 `ramp_step_ms` 和 `ramp_step_type` 属性都需要添加到元组数组中。因此，它们包含 token_ref 节点，其值为 `sof_tkn_volume.word`，指示属性应使用 `word` 类型的 `sof_tkn_volume` 元组数组进行打包：
```
#
# Volume ramp step in milliseconds
#
DefineAttribute."ramp_step_ms" {
        # Token set reference name
        token_ref       "sof_tkn_volume.word"
        constraints {
                min 200
                max 500
        }
}
DefineAttribute."ramp_step_type" {
        type    "string"
        # Token set reference name
        token_ref       "sof_tkn_volume.word"
        constraints {
                !valid_values [
                        "linear"
                        "log"
                        "linear_zc"
                        "log_zc"
                ]
        }
}
```

有时，属性的 `valid_values` 可能需要从人类可读的值转换为整数元组值，以便内核驱动程序可以正确解析它。在上面的示例中，`ramp_step_type` 的有效值被定义为人类可读的字符串值，为 linear 和 log 等。这些值在添加到元组数组之前会被转换为元组值 (0, 1, 等)。
```
DefineAttribute."ramp_step_type" {
        type    "string"
        # Token set reference name
        token_ref       "sof_tkn_volume.word"
        constraints {
                !valid_values [
                        "linear"
                        "log"
                        "linear_zc"
                        "log_zc"
                ]
                !tuple_values [
                        0
                        1
                        2
                        3
                ]
        }
}
```

#### 一个完整的类定义

将所有内容放在一起，以下示例演示了 pga widget 类的完整定义：
```
Class.Widget."pga" {
        # attribute definitions
        DefineAttribute.instance {
                type    integer
        }
        DefineAttribute.index {
                type    integer
        }
        DefineAttribute."type" {
                type    "string"
        }
        DefineAttribute."uuid" {
                type    "string"
                # Token set reference name and type
                token_ref       "sof_tkn_comp.uuid"
        }
        DefineAttribute."preload_count" {}

        #
        # Volume ramp step in milliseconds
        #
        DefineAttribute."ramp_step_ms" {
                # Token set reference name
                token_ref       "sof_tkn_volume.word"
                constraints {
                        min 200
                        max 500
                }
        }
        DefineAttribute."ramp_step_type" {
                type    "string"
                # Token set reference name
                token_ref       "sof_tkn_volume.word"
                constraints {
                        !valid_values [
                                "linear"
                                "log"
                                "linear_zc"
                                "log_zc"
                        ]
                        !tuple_values [
                                0
                                1
                                2
                                3
                        ]
                }
        }

        # attribute qualifiers
        attributes {
                #
                # The PGA widget name is constructed using the index and instance attributes.
                # For ex: "pga.1.1" or "pga.10.2" etc.
                #
                !constructor [
                        "index"
                        "instance"
                ]

                #
                # immutable attributes cannot be modified in the object instance
                #
                !immutable [
                        "uuid"
                        "type"
                ]

                #
                # deprecated attributes should not be added in the object instance
                #
                !deprecated [
                        "preload_count"
                ]

                #
                # pga widget objects instantiated within the same alsaconf node should have
                # unique instance attribute
                #
                unique  "instance"
        }

        # default attribute values
        type            "pga"
        uuid            "7e:67:7e:b7:f4:5f:88:41:af:14:fb:a8:bd:bf:86:82"
        ramp_step_ms    200
}
```

### 对象

对象用于实例化同一类的多个实例，以避免重复的公共属性定义。对象使用新的关键字 `Object` 后根三个节点来实例化：
```
Object.Widget.pga."1" {}
```

这些节点指向以下元素：

 * 对象的类所属的类组。在这个例子中，类属于 `Widget`。
 * 类名称。这里是 `pga`。
 * 唯一的属性值。这是在类定义中，被限定为 `unique` 的属性的值。这里是 `instance`。

使用 [完整的类定义](https://thesofproject.github.io/latest/developer_guides/topology2/topology2.html#complete-class-definition) 中所述的 pga 类定义，你可以通过以下方式实例化一个 pga widget 对象：
```
Object.Widget.pga."1" {
        index 5
}
```

其中 `1` 是 pga 类定义中唯一的属性 `instance` 的值，`index` 属性的值为 5。由于类定义不包含其它的强制性属性，因此上面的实例完全有效。

 > **重要**
 > 你不需要在对象实例中复制公用的属性值。对象自动从其类定义继承属性的默认值。

#### 修改默认属性

类定义中具有默认值的属性，可以通过在对象实例中指定新值来覆盖：
```
Object.Widget.pga."1" {
        index           5
        ramp_step_ms    300
}
```

上面的对象，将类定义中 `ramp_step_ms` 的默认值 200 ms 覆盖为新值 300 ms。

#### 类中的对象

类定义还可以选择包括需要为类对象的每个实例实例化的子对象。例如，pga widget 通常始终包含音量混音器 control。混音器 control 类定义如下：
```
Class.Control."mixer" {
        #
        # Pipeline ID for the mixer object
        #
        DefineAttribute."index" {}

        #
        # Instance of mixer object in the same alsaconf node
        #
        DefineAttribute."instance" {}

        #
        # Mixer name. A mixer object is included in the built topology only if it is given a
        # name
        #
        DefineAttribute."name" {
                type    "string"
        }

        #
        # Max volume setting
        #
        DefineAttribute."max" {}

        DefineAttribute."invert" {
                type    "string"
                constraints {
                        !valid_values [
                                "true"
                                "false"
                        ]
                }
        }

        # use mute LED
        DefineAttribute."mute_led_use" {
                token_ref       "sof_tkn_mute_led.word"
        }

        # LED direction
        DefineAttribute."mute_led_direction" {
                token_ref       "sof_tkn_mute_led.word"
        }

        #
        # access control for mixer
        #
        DefineAttribute."access" {
                type    "compound"
                constraints {
                        !valid_values [
                                "read_write"
                                "tlv_read_write"
                                "read"
                                "write"
                                "volatile"
                                "tlv_read"
                                "tlv_write"
                                "tlv_command"
                                "inactive"
                                "lock"
                                "owner"
                                "tlv_callback"
                        ]
                }
        }

        attributes {
                #
                # The Mixer object name is constructed using the index and instance arguments.
                # For ex: "mixer.1.1" or "mixer.10.2" etc.
                #
                !constructor [
                        "index"
                        "instance"
                ]
                !mandatory [
                        "max"
                ]
                #
                # mixer control objects instantiated within the same alsaconf node should have unique
                # index attribute
                #
                unique  "instance"
        }

        # Default attribute values for mixer control
        invert          "false"
        mute_led_use            0
        mute_led_direction      0
}
```

你可以将混音器 control 对象添加到 pga widget 类定义中：
```
Class.Widget."pga" {
        # Attributes, qualifiers and default values are skipped for simplicity.
        # Refer to the complete class definition in "Complete Class Definition" for details

        # volume control for pga widget
        Object.Control.mixer."1" {
                        name "My Volume Control"
                        max 32
                }
        }
}
```

混音器 control `My Volume Control` 将以编程方式添加到所有 pga 对象中。

#### 对象属性继承

在上面的对象实例化中需要注意的一件事是，混音器对象有两个强制性属性，index 和 instance。但实例化中缺少 index 属性值。这是因为混音器 control 对象在实例化时，从其父 pga 对象继承了 index 属性值。例如，考虑如下的 pga 对象实例：
```
Object.Widget.pga.1 {
        index 5
}
```

pga 类定义中的混音器 control 对象继承索引值 `5`。仅当子对象的类定义与其父类定义共享同名属性时，才会发生继承。对于混音器 control 类和 pga widget 类，共享的属性为 `index`。

#### 设置子对象属性

让我们再次考虑带有混音器 control 对象的 pga 类定义：
```
Class.Widget."pga" {
        # Attributes, qualifiers and default values are skipped for simplicity.
        # Please refer to the complete class definition above for details

        # volume control for pga widget
        Object.Control.mixer."1" {
                        name "My Volume Control"
                        max 32
                }
        }
}
```

请注意，在 pga widget 类定义中，设置了混音器 control 对象的名称。但是，理想情况下，每当实例化新的 pga widget 对象时，我们都希望为混音器 control 指定一个新名称。你可以这样做：
```
Object.Widget.pga."1" {
        index 5

        # volume control'
        Object.Control.mixer."1" {
                        name "My Control Volume 5"
                }
        }
}
```

现在，混音器 control 对象被赋予了名称 `My Control Volume 5`。

#### 嵌套对象

对象还可以实例化为其它对象实例中的子对象。例如，可以在实例化期间将开关 control 添加到 pga widget 对象：
```
Object.Widget.pga."1" {
        index 5

        # volume control
        Object.Control.mixer."1" {
                        name "My Control Volume 5"
                }
        }

        # mute control
        Object.Control.mixer."2" {
                        name "Mute Switch Control"
                        max 1
                }
        }
}
```

请注意两个混音器 control 对象的 `unique` 属性如何不同以保持混音器实例的唯一性。

#### 递归对象属性继承

对象可以嵌套在对象内，而后者又嵌套在其它对象本身内。在这种情况下，属性值可以从顶层父对象一路继承。例如，考虑以下的 volume-playback 流水线的类定义：
```
Class.Pipeline."volume-playback" {
        # Other attributes and qualifiers ommitted for simplicity
        DefineAttribute."index" {}

        DefineAttribute."format" {
                type    "string"
        }

        # pipeline objects
        Object.Widget {
                # Other objects ommitted for simplicity

                pga."1" {}
        }
}
```

请注意，上面的 pga widget 对象没有 index 属性值。volume-playback 类的对象实例化为：
```
Object.Pipeline.volume-playback.1 {
        index 1
        format s24le
}
```

这确保了 volume-playback 对象中的所有子对象都将从它继承 index 属性值。因此 pga widget 对象将具有相同的 index。按照同样的规则，pga widget 对象中的混音器 control 对象也将具有相同的 index 属性值 1。

#### 在父对象树的深处设置子对象属性

在 [设置子对象属性](https://thesofproject.github.io/latest/developer_guides/topology2/topology2.html#setting-child-object-attributes) 中，我们看到我们可以从其父对象实例设置子属性值。例如，你可以从 pga widget 对象实例设置混音器 control 对象的名称。这可以进一步扩展，并且可以从 pga 对象的父对象设置混音器 control 名称。考虑上一节中的 volume-playback 对象实例。我们可以为 pga 对象设置混音器 control 名称，如下所示：
```
Object.Pipeline.volume-playback.1 {
        index 1
        format s24le
        Object.Widget.pga.1 {
                Object.Control.mixer.1 {
                        name    "My Control Volume 1"
                }
        }
}
```

### 顶层配置文件中的参数

参数用于传递构建时参数，它们可用于从同一配置文件构建多个二进制文件。考虑以下具有两个流水线的顶层拓扑配置文件：
```
# arguments
@args [ DYNAMIC_PIPELINE ]
@args.DYNAMIC_PIPELINE {
       type integer
       default 0
}

Object.Pipeline {
        volume-playback.1 {
                dynamic_pipeline $DYNAMIC_PIPELINE
                index 1
                Object.Widget.pipeline.1 {
                        stream_name 'dai.HDA.0.playback'
                }
                Object.Widget.host.playback {
                        stream_name 'Passthrough Playback 0'
                }
                Object.Widget.pga.1 {
                        Object.Control.mixer.1 {
                                name '1 My Playback Volume'
                        }
                }
                format s24le
        }
        volume-playback.3 {
                dynamic_pipeline $DYNAMIC_PIPELINE
                index 3
                Object.Widget.pipeline.1 {
                        stream_name 'dai.HDA.2.playback'
                }
                Object.Widget.host.playback {
                        stream_name 'Passthrough Playback 1'
                }
                Object.Widget.pga.1 {
                        Object.Control.mixer.1 {
                                name '3 My Playback Volume'
                        }
                }
                format s24le
        }
}
```

在这个例子中，volume-playback 对象中的 `dynamic_pipeline` 属性值，从编译拓扑二进制文件时提供的 `-DDYNAMIC_PIPELINE=1` 或 `-DDYNAMIC_PIPELINE=0` 选项给 `DYNAMIC_PIPELINE` 参数的值中扩展。

> **注意**
> alsatplg 编译器仅解析机器拓扑文件中顶层节点定义的参数。

### 包含

构建顶层配置文件时，它应该包含正在实例化的对象的所有类定义，否则编译器将发出错误，指出缺少类定义。所有路径均相对于环境变量 `ALSA_CONFIG_DIR` 指定的目录。你可以像下面这样为依赖指定包含路径：
```
<searchdir:include>
<searchdir:include/controls>
<searchdir:include/components>
```

像下面这样包含类定义：
```
<dai.conf>
<data.conf>
<pcm.conf>
<volume-playback.conf>
```

## 简单的机器拓扑

机器拓扑通常由以下部分组成：

 * 包含路径指向类定义搜索目录的路径
 * Conf 文件 Includes 包含类定义
 * 参数
 * 流水线对象
 * BE DAI 链接对象
 * PCM 对象
 * 顶层流水线连接

让我们考虑一个简单的机器拓扑配置文件，它包含一个音量-播放流水线，一个 HDA 类型的 DAI 链接，一个播放 PCM，和顶层连接：
```
# Include paths
<searchdir:include>
<searchdir:include/common>
<searchdir:include/components>
<searchdir:include/controls>
<searchdir:include/dais>
<searchdir:include/pipelines>

# Include class definitions
<vendor-token.conf>
<tokens.conf>
<volume-playback.conf>
<dai.conf>
<data.conf>
<pcm.conf>
<pcm_caps.conf>
<fe_dai.conf>
<hda.conf>
<hw_config.conf>
<manifest.conf>
<route.conf>

# arguments
@args.DYNAMIC_PIPELINE {
       type integer
       default 0
}

# DAI definition
Object.Dai {
        HDA.0 {
                name 'Analog Playback and Capture'
                id 4
                default_hw_conf_id 4
                Object.Base.hw_config.HDA0 {}
                Object.Widget.dai.1 {
                        direction playback
                        index 1
                        type dai_in
                        stream_name 'Analog Playback and Capture'
                        period_sink_count 0
                        period_source_count 2
                        format s32le
                }
        }
}


# Pipeline Definition
Object.Pipeline {
        volume-playback.1 {
                dynamic_pipeline $DYNAMIC_PIPELINE
                index 1
                Object.Widget.pipeline.1 {
                        stream_name 'dai.HDA.0.playback'
                }
                Object.Widget.host.playback {
                        stream_name 'Passthrough Playback 0'
                }
                Object.Widget.pga.1 {
                        Object.Control.mixer.1 {
                                name '1 My Playback Volume'
                        }
                }
                format s24le
        }
}

# PCM Definitions
Object.PCM {
        pcm.0 {
                name 'HDA Analog'
                Object.Base.fe_dai.'HDA Analog' {}
                Object.PCM.pcm_caps.playback {
                        name 'Passthrough Playback 0'
                        formats 'S24_LE,S16_LE'
                }
                direction playback
                id 0
        }
}

# Top-level pipeline connection
# Buffer.1. -> dai.HDA.1.playback
Object.Base.route.1 {
        source 'buffer.1.1'
        sink 'dai.HDA.1.playback'
}
```

注意上面的配置文件只包含，volume-playback 流水线中的缓冲区 widget `buffer.1.1` 和 dai widget `dai.HDA.1.playback` 之间的顶层路由。volume-playback 流水线中的 widgets 之间的连接在类定义中定义。

让我们深入了解一下 volume-playback 流水线的类定义，看看类定义中包含的路由对象。有关完整的类定义，请参阅 [volume-playback](https://github.com/thesofproject/sof/blob/main/tools/topology/topology2/include/pipelines/volume-playback.conf)。
```
Class.Pipeline."volume-playback" {
        # pipeline attributes skipped for simplicity

        attributes {
                # pipeline name is constructed as "volume-playback.1"
                !constructor [
                        "index"
                ]
                !mandatory [
                        "format"
                ]
                !immutable [
                        "direction"
                ]
                #
                # volume-playback objects instantiated within the same alsaconf node should have
                # unique instance attribute
                #
                unique  "instance"
        }

        # Widget objects that constitute the volume-playback pipeline
        Object.Widget {
                pipeline."1" {}

                host."playback" {
                        type            "aif_in"
                }

                buffer."1" {
                        periods 2
                        caps            "host"
                }

                pga."1" {
                        Object.Control.mixer.1 {
                                Object.Base.tlv."vtlv_m64s2" {
                                        Object.Base.scale."m64s2" {}
                                }
                        }
                }

                buffer."2" {
                        periods 2
                        caps            "dai"
                }
        }

        # Pipeline connections.
        # The index attribute values for the source/sink widgets will be populated
        # when the route objects are built
        Object.Base {
                route."1" {
                        source  "host..playback"
                        sink    "buffer..1"
                }

                route."2" {
                        source  "buffer..1"
                        sink    "pga..1"
                }

                route."3" {
                        source  "pga..1"
                        sink    "buffer..2"
                }
        }

        # Default attribute values
        direction       "playback"
        time_domain     "timer"
        period          1000
        channels        2
        rate            48000
        priority        0
        core            0
        frames          0
        mips            5000
}
```

除了路由对象实例之外，流水线类定义相当容易理解。我们再进一步分析一下。 路由类定义定义如下：
```
Class.Base."route" {
        # sink widget name
        DefineAttribute."sink" {
                type    "string"
        }

        # source widget name for route
        DefineAttribute."source" {
                type    "string"
        }

        # control name for the route
        DefineAttribute."control" {
                type    "string"
        }

        #
        # Pipeline ID of the pipeline the route object belongs to
        #
        DefineAttribute."index" {}

        # unique instance for route object in the same alsaconf node
        DefineAttribute."instance" {}

        attributes {
                !constructor [
                        "instance"
                ]
                !mandatory [
                        "source"
                        "sink"
                ]
                #
                # route objects instantiated within the same alsaconf node should have unique
                # index attribute
                #
                unique  "instance"
        }
}
```

请注意，路由对象应具有 instance、source 和 sink 属性。

让我们再次考虑 volume-playback 类中的路由对象：
```
Object.Base {
        route."1" {
                source  "host..playback"
                sink    "buffer..1"
        }

        route."2" {
                source  "buffer..1"
                sink    "pga..1"
        }

        route."3" {
                source  "pga..1"
                sink    "buffer..2"
        }
}
```

请注意，source 和 sink 属性是为所有路由定义的。比如，第二个路由对象 `Object.Base.route.2`，其 sink 属性值为 `pga..1`。参阅 [一个简单的类定义](https://thesofproject.github.io/latest/developer_guides/topology2/topology2.html#complete-class-definition) 中的 pga widget 类定义，我们知道 pga widget 对象的构造函数具有两个属性，`index` 和 `instance`。通过查看 widgets 列表，我们知道 volume-playback 类中的 pga widget 实例为 1。但流水线中的 pga widget 的 index 属性值未知。它只能从顶级拓扑配置文件中设置，如 [简单机器拓扑](https://thesofproject.github.io/latest/developer_guides/topology2/topology2.html#simple-machine-topology) 中所示。因此，在类定义中，index 属性留空。当构建路由对象时，alsatplg 编译器将使用适当的值填充 index 属性。对于上面的机器拓扑，将使用正确的流水线 ID 构建路由对象 `Object.base.route.2`，如下所示：
```
Object.base.route.2 {
        source  "buffer.1.1"
        sink "pga.1.1"
}
```

目前，alsatplg 只能为路由对象 source 和 sink 属性填写属性值。如果需要，可以将此功能扩展到其它类型的对象。

## 条件包含

条件包含允许从同一输入配置文件构建多个拓扑二进制文件。例如，让我们考虑 HDA 通用机器拓扑。DMIC 的数量决定是否应包含 DMIC 配置文件。这可以通过以下方式实现：
```
@args.DMIC_COUNT {
       type integer
       default 0
}

# include DMIC config if needed
IncludeByKey.DMIC_INCLUDE {
        "[1-4]" "include/platform/intel/dmic-generic.conf"
}
```

正则表达式 `[1-4]` 指示，如果 DMIC_COUNT 参数值在 1 到 4 之间，则应包含 dmic-generic.conf 文件。假设顶层文件名为 `sof-hda-generic.conf`，你可以使用以下命令构建两个单独的拓扑二进制文件：

 * 对于没有 DMIC 的机器：
```
alsatplg -p -c sof-hda-generic.conf -o sof-hda-generic.tplg
```

 * 对于具有两个 DMIC 的机器：
```
alsatplg -D DMIC_COUNT=2 -p -c sof-hda-generic.conf -o sof-hda-generic-2ch.tplg
```

条件包含不仅限于顶层配置文件。你可以将它们添加到配置文件中的任何节点，以在指定节点处包含配置。例如，我们可以为 EQIIR widget 的 byte controls 条件包含正确的过滤器系数。

在顶层拓扑文件中为系数定义参数：
```
@args.EQIIR_BYTES {
       type string
       default "highpass_40hz_0db_48khz"
}
```

然后包含系数：
```
Object.Widget.eqiir.1 {
        Object.Control.bytes.1 {
                name "my eqiir byte control"
                # EQIIR filter coefficients
                IncludeByKey.EQIIR_BYTES {
                        "[highpass.40hz.0db.48khz]" "include/components/eqiir/highpass_40hz_0db_48khz.conf"
                        "[highpass.40hz.20db.48khz]" "include/components/eqiir/highpass_40hz_20db_48khz.conf"
                }
        }
}
```

## 构建 2.0 配置文件

你可以使用 alsatplg 编译拓扑 2.0 配置文件并生成拓扑二进制文件：
```
alsatplg <-D args=values> -p -c input.conf -o output.tplg
```

`-D` 开关用于给顶层配置文件传递逗号分隔的参数值。

你可以使用 `-P` 开关将 2.0 配置文件转为 1.0 配置文件：
```
alsatplg <-D args=values> -P input.conf -o output.conf
```

## 拓扑提醒

查看以下拓扑注意事项：

 * “index” 指流水线，widget 和 control 类组中的流水线 ID。
 * DAI 类组对象中的 “id” 指的是内核中机器驱动程序中定义的链接 ID。

## Alsaconf 提醒

查看以下 alsaconf 注意事项：

 * “.” 指节点分割符。“foo.bar value” 相当于以下内容：
```
foo {
        bar value
}
```
 * 数组用 [] 定义。例如：
```
!constructor [
        "foo"
        "bar"
]
```

我们建议在类定义中的数组定义中使用感叹号 (!)。如果从不同源多次包含类配置文件，则使用它可以确保数组项不重复。

[原文](https://thesofproject.github.io/latest/developer_guides/topology2/topology2.html)

Done.
