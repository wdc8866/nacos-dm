# 多架构 Nacos + DM 插件镜像
# 构建后推送到阿里云 ACR，支持 linux/amd64 和 linux/arm64

FROM nacos/nacos-server:v3.2.4

# 复制 DM 插件包（含 base 模块、12 个 mapper、dialect、schema、sql-error-codes）
# shaded jar 已内置 DmJdbcDriver18，但建议再单独放一份驱动，防止类加载顺序问题
COPY nacos-dm-datasource-plugin-ext.jar /home/nacos/plugins/
COPY DmJdbcDriver18.jar /home/nacos/plugins/

# 可选：若需自定义启动参数，可取消下面注释并按需修改
# ENV JAVA_OPT_EXT="-Dnacos.plugin.datasource.log.enabled=true"