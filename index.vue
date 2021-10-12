<template>
  <div>
    <div
      class="container"
      id="treecontainer"
      :style="{ height: height + 'px' }"
    >
      <div class="menu">
        <div id="product_tree" class="treeWrap"></div>
      </div>
    </div>
  </div>
</template>
<script>
import { getCompanyShareholder } from "@/api/equityPenetrationChart/index";
import d3 from "./d3.min.js";
export default {
  name: "equityPenetrationChart",
  props: {
    companyCreditCode: {
      type: String,
      default: "",
    },
    companyName: {
      type: String,
      default: "",
    },
    loading: {
      type: Boolean,
      default: false,
    },
    height: {
      type: Number,
      default: 420,
    },
  },
  data() {
    return {
      container: "",
      zoom: "",
      rootData: {
        upward: {
          direction: "upward",
          name: "origin",
          children: [],
        },
        downward: {
          direction: "downward",
          name: "origin",
          children: [],
        },
      },
      depthInfo: "",
      rootName: "",
      type: 0,
    };
  },
  methods: {
    getData() {
      let params = {
        companyCreditCode: this.companyCreditCode,
        companyName: this.companyName,
        type: this.type,
        companyCode: this.$route.query.companyCode,
      };
      this.$emit("updateLoading", true);
      getCompanyShareholder(params).then((res) => {
        console.log(res, "企业详情--股权穿透接口");
        this.$emit("updateLoading", false);
        if (res.code == 1) {
          let downList = res.retInfo.downward;
          let upList = res.retInfo.upward;
          if (downList.length == 0 && upList.length == 0) {
            this.rootName = res.retInfo.main.name;

            this.$emit("hasData", true);
            this.drawing();
          } else {
            this.rootData.downward.children = downList;
            this.rootData.upward.children = upList;
            this.rootName = res.retInfo.main.name;
            this.$emit("hasData", true);

            this.drawing();
          }
        }
      });
      // rootName = "上海冰鉴信息科技有限公司";
    },

    drawing() {
      var _this = this;
      var clickAble = true;
      // var rootName = ''; //根节点的名字
      var rootRectWidth = 0; //根节点rect的宽度
      var downwardLength = 0,
        upwardLength = 0;
      var forUpward = true;
      var width = document.getElementById("product_tree").scrollWidth;
      var height =
        document.getElementById("product_tree").scrollHeight || this.height;
      var treeChart = function (d3Object) {
        this.d3 = d3Object;
        this.directions = ["upward", "downward"];
      };

      treeChart.prototype.drawChart = function () {
        // First get tree data for both directions.

        this.treeData = {};
        var self = this;
        self.directions.forEach(function (direction) {
          self.treeData[direction] = _this.rootData[direction];
        });
        rootRectWidth =
          _this.rootName.length * 15 < 140 ? 140 : _this.rootName.length * 15;
        //获得upward第一级节点的个数
        upwardLength = _this.rootData.upward.children.length;
        //获得downward第一级节点的个数
        downwardLength = _this.rootData.downward.children.length;
        self.graphTree(self.getTreeConfig());
      };

      treeChart.prototype.getTreeConfig = function () {
        var treeConfig = {
          margin: {
            top: 10,
            right: 5,
            bottom: 0,
            left: 5,
          },
        };

        treeConfig.chartWidth =
          width - treeConfig.margin.right - treeConfig.margin.left;
        treeConfig.chartHeight =
          height - treeConfig.margin.top - treeConfig.margin.bottom;
        treeConfig.centralHeight = treeConfig.chartHeight / 2;
        treeConfig.centralWidth = treeConfig.chartWidth / 2;
        treeConfig.linkLength = 90; //箭头长度(上下节点距离)
        treeConfig.duration = 500; //动画时间
        return treeConfig;
      };

      treeChart.prototype.graphTree = function (config) {
        var self = this;
        var d3 = this.d3;
        var linkLength = config.linkLength;
        var duration = config.duration;
        var hasChildNodeArr = [];
        var id = 0;
        var diagonal = function (obj) {
          //折线
          // console.log(obj);
          var s = obj.d.source;
          var t = obj.d.target;
          var endMoveNum = 0;
          var moveDistance = 0;
          if (obj.d) {
            if (obj.d.source.direction == "downward") {
              var downMoveNum = obj.d.source.name == "origin" ? 10 : 30;
              var tmpNum = s.y + (t.y - s.y) / 2;
              endMoveNum = downMoveNum;
              moveDistance = tmpNum + endMoveNum;
            } else {
              var upMoveNum = obj.d.source.name == "origin" ? -10 : -30;
              var tmpNum = t.y + (s.y - t.y) / 2;
              endMoveNum = upMoveNum;
              moveDistance = tmpNum + endMoveNum;
            }
          }
          return (
            "M" +
            s.x +
            "," +
            (s.y + (s.direction == "downward" ? 10 : -11)) +
            "L" +
            s.x +
            "," +
            moveDistance +
            "L" +
            t.x +
            "," +
            moveDistance +
            "L" +
            t.x +
            "," +
            (t.y + (t.direction == "upward" ? -11 : 2))
          );
        };
        // var zoom = d3.behavior.zoom().scaleExtent([1, 1]).on("zoom", redraw);
        var svg = d3
          .select("#product_tree")
          .append("svg")
          .attr(
            "width",
            config.chartWidth + config.margin.right + config.margin.left
          )
          .attr(
            "height",
            config.chartHeight + config.margin.top + config.margin.bottom
          )
          .attr("xmlns", "http://www.w3.org/2000/svg")
          // .on("mousedown", disableRightClick)
          // .call(zoom)
          .on("dblclick.zoom", null);
        var treeG = svg
          .append("g")
          .attr("class", "gbox")
          .attr(
            "transform",
            "translate(" + config.margin.left + "," + config.margin.top + ")"
          );

        //箭头(下半部分)
        var markerDown = svg
          .append("marker")
          .attr("id", "resolvedDown")
          .attr("markerUnits", "strokeWidth") //设置为strokeWidth箭头会随着线的粗细发生变化
          .attr("markerUnits", "userSpaceOnUse")
          .attr("viewBox", "0 -5 10 10") //坐标系的区域
          .attr("refX", 0) //箭头坐标
          .attr("refY", 0)
          .attr("markerWidth", 10) //标识的大小
          .attr("markerHeight", 10)
          .attr("orient", "90") //绘制方向，可设定为：auto（自动确认方向）和 角度值
          .attr("stroke-width", 2) //箭头宽度
          .append("path")
          .attr("d", "M0,-5L10,0L0,5") //箭头的路径
          .attr("fill", "#128bed")
          .attr("fill-opacity", 1); //箭头颜色
        //箭头(上半部分)
        var markerUp = svg
          .append("marker")
          .attr("id", "resolvedUp")
          .attr("markerUnits", "strokeWidth") //设置为strokeWidth箭头会随着线的粗细发生变化
          .attr("markerUnits", "userSpaceOnUse")
          .attr("viewBox", "0 -5 10 10") //坐标系的区域
          .attr("refX", -26) //箭头坐标
          .attr("refY", 0)
          .attr("markerWidth", 10) //标识的大小
          .attr("markerHeight", 10)
          .attr("orient", "90") //绘制方向，可设定为：auto（自动确认方向）和 角度值
          .attr("stroke-width", 2) //箭头宽度
          .append("path")
          .attr("d", "M0,-5L10,0L0,5") //箭头的路径
          .attr("fill", "#128bed")
          .attr("fill-opacity", 1); //箭头颜色

        // Initialize the tree nodes and update chart.
        for (var d in this.directions) {
          var direction = this.directions[d];
          var data = self.treeData[direction];
          data.x0 = config.centralWidth;
          data.y0 = config.centralHeight;
          data.children.forEach(collapse);
          update(data, data, treeG);
        }

        function update(source, originalData, g) {
          // console.log(source);
          var direction = originalData["direction"];
          forUpward = direction == "upward";
          var node_class = direction + "Node";
          var link_class = direction + "Link";
          var downwardSign = forUpward ? -1 : 1;
          // var nodeColor = forUpward ? "#128bed" : "#8b4513";

          var isExpand = false;
          // var statusUp = true;
          // var statusDown = true;
          var nodeSpace = 130; //左右节点间距
          var tree = d3.layout.tree().sort(sortByDate).nodeSize([nodeSpace, 0]);
          var nodes = tree.nodes(originalData);
          var links = tree.links(nodes);
          var offsetX = -config.centralWidth;
          nodes.forEach(function (d) {
            d.y = downwardSign * (d.depth * linkLength) + config.centralHeight;
            d.x = d.x - offsetX;
            if (d.name == "origin") {
              d.x = config.centralWidth;
              d.y += downwardSign * 0; // 上下两树图根节点之间的距离
            } else if (d.parent.name == "origin") {
              d.y =
                downwardSign * (d.depth * linkLength) +
                config.centralHeight -
                (forUpward ? -15 : 15); //根节点距离子节点的距离
            } else if (d.depth >= 1) {
              d.y =
                downwardSign * (d.depth * linkLength) +
                config.centralHeight -
                (forUpward ? (d.depth - 2) * 18 : -((d.depth - 2) * 18));
            }
          });

          // Update the node.
          var node = g.selectAll("g." + node_class).data(nodes, function (d) {
            return d.id || (d.id = ++id);
          });
          var nodeEnter = node
            .enter()
            .append("g")
            .attr("class", node_class)
            .attr("transform", function (d) {
              return "translate(" + source.x0 + "," + source.y0 + ")";
            })
            .style("cursor", function (d) {
              return d.name == "origin"
                ? ""
                : d.children || d._children
                ? "pointer"
                : "";
            })
            .on("click", nodeClick)
            .on("mouseenter", nodeHover)
            .on("mouseleave", nodeOut);
          nodeEnter
            .append("svg:rect")
            .attr("x", function (d) {
              return d.name == "origin" ? -(rootRectWidth / 2) : -60;
            })
            .attr("y", function (d) {
              return d.name == "origin" ? -20 : forUpward ? -52 : 12;
            })
            .attr("width", function (d) {
              return d.name == "origin" ? rootRectWidth : 120;
            })

            .attr("height", 40)
            // .attr("rx", 10)//节点边框圆角
            .style("stroke", function (d) {
              return d.name == "origin"
                ? "#128bed"
                : d.type == 2
                ? "#128bed"
                : "#FF4D4D"; //节点边框
            })
            .style("fill", function (d) {
              return d.name == "origin" ? "#128bed" : "#FFF"; //节点背景色
            })
            .on("mouseenter", nodeHover);

          nodeEnter
            .append("circle")
            .attr("r", 1e-6)
            .on("mouseenter", nodeOut)
            .on("mouseleave", nodeOut);
          nodeEnter
            .append("text")
            .attr("class", "linkname")
            .attr("x", function (d) {
              return d.name == "origin" ? "0" : "0";
            })
            .attr("dy", function (d) {
              return d.name == "origin"
                ? ".35em"
                : forUpward
                ? d.name.length > 10
                  ? "-34"
                  : "-28"
                : d.name.length > 10
                ? "30"
                : "35";
            })
            .attr("text-anchor", function (d) {
              return d.name == "origin" ? "middle" : "start";
            })
            .attr("fill", "#000")
            .text(function (d) {
              if (d.name == "origin") {
                return _this.rootName;
              }
              if (d.repeated) {
                return "[Recurring] " + d.name;
              }
              return d.name.length > 10 ? d.name.substr(0, 10) : d.name;
            })
            .style({
              fill: function (d) {
                if (d.name == "origin") {
                  return "#fff";
                }
              },
              "font-size": function (d) {
                return d.name == "origin" ? 14 : 11;
              },
            });

          nodeEnter
            .append("text")
            .attr("class", "linkname")
            .attr("dy", function (d) {
              return d.name == "origin" ? ".35em" : forUpward ? "-20" : "43";
            })
            .attr("text-anchor", function () {
              return d.name == "origin" ? "middle" : "start"; //字体上下居中？
            })
            .text(function (d) {
              if (d.name.substr(10, d.name.length).length > 10) {
                return d.name.slice(10, 19) + "...";
              } else {
                return d.name.substr(10, d.name.length);
              }
            })
            .style({
              // 'fill': "#128bed",第二行字体颜色
              "font-size": function (d) {
                return d.name == "origin" ? 14 : 11;
              },
            });
          // .on('click', Change_modal);

          // nodeEnter
          //   .append("text")
          //   .attr("x", "-55")
          //   .attr("dy", function (d) {
          //     return d.name == "origin" ? ".35em" : forUpward ? "-16" : "48";
          //   })
          //   .attr("text-anchor", "start")
          //   .attr("class", "linkname")
          //   .style("fill", "#128bed") //认缴金额字体颜色
          //   .style("font-size", 10)
          //   .text(function (d) {
          //     var str =
          //       d.name == "origin" ? "" : "认缴金额:" + d.amount + "万人民币";
          //     return str.length > 13 ? str.substr(0, 13) + ".." : str;
          //   });
          nodeEnter
            .append("text")
            .attr("x", "25")
            .attr("dy", function (d) {
              return d.name == "origin" ? ".35em" : forUpward ? "2" : "-12";
            })
            .attr("text-anchor", "start")
            .attr("class", "linkname")
            .style("fill", "#128bed") //比例颜色（55%）
            .style("font-size", 10)
            .text(function (d) {
              if (d.ratio != "0.00%" && d.ratio != "") {
                return d.name == "origin" ? "" : d.ratio;
              }
            })
            .on("mouseenter", nodeOut)
            .on("mouseleave", nodeOut);

          // Transition nodes to their new position.原有节点更新到新位置
          var nodeUpdate = node
            .transition()
            .duration(duration)
            .attr("transform", function (d) {
              return "translate(" + d.x + "," + d.y + ")";
            });
          nodeUpdate
            .select("circle")
            .attr("r", function (d) {
              return d.name == "origin"
                ? 0
                : hasChildNodeArr.indexOf(d) == -1
                ? 0
                : 6;
            })
            .attr("cy", function (d) {
              return d.name == "origin" ? -20 : forUpward ? -59 : 59;
            })
            .style("fill", function (d) {
              return hasChildNodeArr.indexOf(d) != -1 ? "#fff" : ""; //展开按钮背景颜色
              // if (d._children || d.children) { return "#fff"; } else { return "rgba(0,0,0,0)"; }
            })
            .style("stroke", function (d) {
              return hasChildNodeArr.indexOf(d) != -1
                ? d.type == 2
                  ? "#128bed"
                  : "#FF4D4D"
                : ""; //展开按钮边框颜色
            })
            .style("stroke-width", function (d) {
              if (d.repeated) {
                return 5;
              }
            });
          //代表是否展开的+-号
          nodeEnter
            .append("svg:text")
            .attr("class", "isExpand")
            .attr("x", "0")
            .attr("dy", function (d) {
              return forUpward ? -58 : 59.5;
            })
            .attr("text-anchor", "middle")
            .style("fill", function (d) {
              return d.type == 2 ? "#128bed" : "#FF4D4D";
            }) //+、-字体颜色
            .text(function (d) {
              if (d.name == "origin") {
                return "";
              }
              return hasChildNodeArr.indexOf(d) != -1 ? "+" : "";
            })
            .on("click", click)
            .on("mouseenter", nodeOut)
            .on("mouseleave", nodeOut);

          var nodeExit = node
            .exit()
            .transition()
            .duration(duration)
            .attr("transform", function (d) {
              return "translate(" + source.x + "," + source.y + ")";
            })
            .remove();
          nodeExit.select("circle").attr("r", 1e-6);
          var link = g
            .selectAll("path." + link_class)
            .data(links, function (d) {
              return d.target.id;
            });
          link
            .enter()
            .insert("path", "g")
            .attr("class", link_class)
            .attr("stroke", function (d) {
              return "#bbb";
            })
            .attr("fill", "none")
            .attr("stroke-width", "1px")
            .attr("opacity", 0.5)
            .attr("d", function (d) {
              var o = {
                x: source.x0,
                y: source.y0,
              };
              return diagonal({
                source: o,
                target: o,
                d,
              });
            })
            .attr("marker-end", function (d) {
              return forUpward ? "url(#resolvedUp)" : "url(#resolvedDown)";
            }) //根据箭头标记的id号标记箭头;
            .attr("id", function (d, i) {
              return "mypath" + i;
            });
          link
            .transition()
            .duration(duration)
            .attr("d", function (d) {
              return diagonal({ d });
            });
          link
            .exit()
            .transition()
            .duration(duration)
            .attr("d", function (d) {
              var o = {
                x: source.x,
                y: source.y,
              };
              return diagonal({
                source: o,
                target: o,
                d,
              });
            })
            .remove();
          nodes.forEach(function (d) {
            d.x0 = d.x;
            d.y0 = d.y;
          });
          function nodeClick(d) {
            console.log(d);
          }
          //添加动态关系线
          function nodeHover(d, i) {
            if (d.name != "origin") {
              // console.log(d, "dddddddddddddddddd");
              if (d.parent.direction == "downward") {
                var links = d3.selectAll(".downwardLink")[0];
                //当前节点的子级节点
                links.map((item, index) => {
                  if (item.__data__.source.id == d.id) {
                    if (d.children) {
                      d.children.forEach((citem) => {
                        if (item.__data__.target.id == citem.id) {
                          d3.select(item).attr(
                            "class",
                            "downwardLink downLine"
                          );
                        }
                      });
                    }
                  } else if (
                    item.__data__.source.id == d.parent.id &&
                    item.__data__.target.id == d.id
                  ) {
                    d3.select(item).attr("class", "downwardLink downLine");
                  }
                });
                //递归处理当前节点的祖先节点
                function changeUpLink(d) {
                  links.map((item, index) => {
                    if (d.name != "origin") {
                      if (
                        item.__data__.source.id == d.parent.id &&
                        item.__data__.target.id == d.id
                      ) {
                        d3.select(item).attr("class", "downwardLink downLine");
                      }
                    }
                  });
                  if (d.depth > 1) {
                    if (!d.parent) {
                      return;
                    } else {
                      changeUpLink(d.parent);
                    }
                  }
                }
                // changeUpLink(d, true);
              } else {
                var links = d3.selectAll(".upwardLink")[0];
                for (let i = 0; i < links.length; i++) {
                  let item = links[i];
                  if (item.__data__.source.id == d.id) {
                    if (d.children) {
                      d.children.forEach((citem) => {
                        if (item.__data__.target.id == citem.id) {
                          d3.select(item).attr("class", "upwardLink upLine");
                        }
                      });
                    }
                  } else if (
                    item.__data__.source.id == d.parent.id &&
                    item.__data__.target.id == d.id
                  ) {
                    d3.select(item).attr("class", "upwardLink upLine");
                  }
                }
                function cancelUpLink(d) {
                  for (let i = 0; i < links.length; i++) {
                    let item = links[i];
                    if (d.name != "origin") {
                      if (
                        item.__data__.source.id == d.parent.id &&
                        item.__data__.target.id == d.id
                      ) {
                        d3.select(item).attr("class", "upwardLink upLine");
                      }
                    }
                  }
                  if (d.parent) {
                    cancelUpLink(d.parent);
                  }
                }
                // cancelUpLink(d);
              }
            }
          }
          function nodeOut(d, i) {
            if (d.name != "origin") {
              if (d.parent.direction == "downward") {
                var links = d3.selectAll(".downwardLink")[0];
                for (let i = 0; i < links.length; i++) {
                  let item = links[i];
                  if (
                    d3.select(item).attr("class").indexOf("downLine") != "-1"
                  ) {
                    d3.select(item).attr("class", "downwardLink");
                  }
                }
              } else {
                var links = d3.selectAll(".upwardLink")[0];
                for (let i = 0; i < links.length; i++) {
                  let item = links[i];
                  if (d3.select(item).attr("class").indexOf("upLine") != "-1") {
                    d3.select(item).attr("class", "upwardLink");
                  }
                }
              }
            }
          }
          function Change_modal() {
            _this.Modal = true;
          }

          function click(d) {
            if (clickAble) {
              isExpand = !isExpand;
              if (d.name == "origin") {
                return;
              }
              clickAble = false;
              if (d.children) {
                clickAble = true;
                d._children = d.children;
                d.children = null;
                update(d, originalData, g);
                d3.select(this)
                  .text("+")
                  .attr("dy", function (d) {
                    return d.direction == "upward" ? -58 : 59.5;
                  });
              } else {
                let type = d.direction == "upward" ? "1" : "2"; //1:向上；2：向下
                let params = {
                  type: type,
                  companyCreditCode: d.companyCreditCode,
                  companyName: d.name,
                  companyCode: d.companyCode,
                };
                let list = [];
                _this.$emit("updateLoading", true);
                getCompanyShareholder(params)
                  .then((res) => {
                    console.log(res, "企业详情--股权穿透接口");
                    clickAble = true;
                    _this.$emit("updateLoading", false);
                    if (res.code == 1) {
                      if (d.direction == "upward") {
                        list = res.retInfo.upward;
                      } else {
                        list = res.retInfo.downward;
                      }
                      d.children = list;
                      d.children.forEach(collapse);
                      d._children = null;
                      // expand all if it's the first node
                      if (d.name == "origin") {
                        d.children.forEach(expand);
                      }
                      d3.select(this)
                        .text("-")
                        .attr("dy", function (d) {
                          return d.direction == "upward" ? -59 : 59;
                        });
                      update(d, originalData, g);
                      let y =
                        d.direction == "upward"
                          ? nodes[0].y - d.y
                          : nodes[0].y - d.y;
                      let x =
                        d.direction == "upward"
                          ? nodes[0].x - d.x
                          : nodes[0].x - d.x;
                      treeG
                        .transition()
                        .duration(duration)
                        .attr("transform", "translate(" + x + "," + y + ")");
                    }
                  })
                  .catch((error) => {
                    console.log(error);
                    clickAble = true;
                  });
              }
            }
          }
        }
        function expand(d) {
          if (d._children) {
            d.children = d._children;
            d.children.forEach(expand);
            d._children = null;
          }
        }

        function collapse(d) {
          if (d.status == 1) {
            d._children = d.children;
            d._children.forEach(collapse);
            d.children = null;
            hasChildNodeArr.push(d);
          }
        }

        // function redraw() {
        //   if (d3.event.sourceEvent.type == "wheel") return;
        //   treeG.attr("transform", "translate(" + d3.event.translate + ")");
        // }

        function disableRightClick() {
          // stop zoom
          if (d3.event.button == 2) {
            // console.log("No right click allowed");
            d3.event.stopImmediatePropagation();
          }
        }

        function sortByDate(a, b) {
          var aNum = a.name.substr(a.name.lastIndexOf("(") + 1, 4);
          var bNum = b.name.substr(b.name.lastIndexOf("(") + 1, 4);
          return (
            d3.ascending(aNum, bNum) ||
            d3.ascending(a.name, b.name) ||
            d3.ascending(a.id, b.id)
          );
        }
      };

      var d3GenerationChart = new treeChart(d3);
      d3GenerationChart.drawChart();
      this.$nextTick(() => {
        var obox = document.getElementById("product_tree").childNodes[0];
        var gbox =
          document.getElementById("product_tree").childNodes[0].childNodes[0];
        obox.addEventListener("mousedown", function (evt) {
          var oEvent = evt || event; //获取事件对象，这个是兼容写法
          var disX = oEvent.clientX;
          var disY = oEvent.clientY;
          let arr = gbox
            .getAttribute("transform")
            .replace("translate(", "")
            .replace(")", "")
            .split(",");

          //这里就解释为什么要给document添加onmousemove时间，原因是如果你给obox添加这个事件的时候，当你拖动很快的时候就很快脱离这个onmousemove事件，而不能实时拖动它
          document.onmousemove = function (evt) {
            //实时改变目标元素obox的位置
            var oEvent = evt || event;
            gbox.setAttribute(
              "transform",
              `translate(${oEvent.clientX - disX + Number(arr[0])},${
                oEvent.clientY - disY + Number(arr[1])
              })`
            );
          };
          //停止拖动
          document.onmouseup = function () {
            document.onmousemove = null;
            document.onmouseup = null;
          };
        });
      });
    },
  },
  mounted() {
    var child = document.getElementById("product_tree");
    child.innerHTML = "";
    this.getData();
  },
};
</script>

<style lang="scss" scoped>
.container {
  box-sizing: border-box;
  width: 100%;
  overflow: hidden;
  background: #fff;
  /* line-height: 20px; */
}

.treeWrap {
  display: flex;
  align-items: center;
  justify-content: center;
}

/* .form_input {
      width: @width;
  } */

svg {
  cursor: all-scroll;
}

.centralText {
  font: 23spx sans-serif;
  fill: #222;
}

.downwardNode text,
.upwardNode text {
  font: 10px sans-serif;
}

.downwardLink {
  fill: none;
  stroke-width: 1px;
  /* opacity: 0.5; */
}

.upwardLink {
  fill: none;
  stroke-width: 1px;
  /* opacity: 0.5; */
}
/deep/.downLine {
  stroke: #128bed;
  stroke-dasharray: 6, 2;
  stroke-dashoffset: 20;
  animation: 0.7s down-lines infinite;
  z-index: 999;
  opacity: 1;
  stroke-width: 2px;
}
/deep/.upLine {
  stroke: #128bed;
  stroke-dasharray: 6, 2;
  stroke-dashoffset: 20;
  animation: 0.7s up-lines infinite;
  z-index: 999;
  opacity: 1;
  stroke-width: 2px;
}
@keyframes down-lines {
  0% {
    stroke-dashoffset: 100;
  }

  100% {
    stroke-dashoffset: -100;
  }
  0% {
    stroke-dashoffset: 100;
  }
}
@keyframes up-lines {
  0% {
    stroke-dashoffset: -100;
  }

  100% {
    stroke-dashoffset: 100;
  }
  0% {
    stroke-dashoffset: 1100;
  }
}
/deep/.isExpand {
  dominant-baseline: middle;
  text-anchor: middle;
}

/deep/.linkname {
  text-anchor: middle;
}
</style>