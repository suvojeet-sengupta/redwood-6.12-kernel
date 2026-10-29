# SPDX-License-Identifier: GPL-2.0

ccflags-y += -I$(src)

# Subdirectories
obj-y += qcom/

obj-$(CONFIG_QTI_BATTERY_CHARGER) += qti-battery-charger.o
qti-battery-charger-objs += qti_battery_charger.o qti_battery_charger_xiaomi.o qti_charger_boost_lib.o qti_typec_class.o
CFLAGS_qti_battery_charger.o += -DCONFIG_BQ_FG_1S -DCONFIG_BQ2597X -DCONFIG_BQ_CLOUD_AUTHENTICATION -DCONFIG_BQ_FUEL_GAUGE -DCONFIG_REDWOOD_FOR_BUILD -DCONFIG_AI_RSOC_M20
CFLAGS_qti_battery_charger_xiaomi.o += $(CFLAGS_qti_battery_charger.o)

obj-$(CONFIG_BATTERY_MAX1720X) += max1720x_battery.o

