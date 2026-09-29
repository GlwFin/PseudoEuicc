package com.example.pseudoeuicc;

import java.lang.reflect.Method;
import java.util.Set;

import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.XC_MethodHook;
import de.robv.android.xposed.XposedBridge;
import de.robv.android.xposed.XposedHelpers;
import de.robv.android.xposed.callbacks.XC_LoadPackage;

/**
 * PseudoEuicc 1.0
 *
 * 将指定卡槽（TARGET_SLOT）伪装为内置 eUICC，使 ColorOS 的 eSIM
 * 设置入口可用。卡的真实 EID 在 hook 运行时自动从 modem 层获取，
 * 无需配置界面。
 *
 * 需 LSPosed，作用域：com.android.phone
 */
public class Main implements IXposedHookLoadPackage {

    private static final String TAG = "PseudoEuicc";

    private static final int TARGET_SLOT = 1;

    private static final String CLASS_UICC_SLOT =
            "com.android.internal.telephony.uicc.UiccSlot";
    private static final String CLASS_UICC_CONTROLLER =
            "com.android.internal.telephony.uicc.UiccController";

    private static String sEid = null;

    private static void log(String m) {
        XposedBridge.log(TAG + ": " + m);
    }

    /** 日志里不打印完整 EID，掩码显示。 */
    private static String mask(String eid) {
        if (eid == null) return "null";
        int n = eid.length();
        if (n <= 8) return "****";
        return eid.substring(0, 4) + "****" + eid.substring(n - 4);
    }

    private static String eidFromEuiccCard(Object slot) {
        try {
            Object card = XposedHelpers.callMethod(slot, "getUiccCard", new Object[0]);
            if (card == null) return null;
            Object eidObj = XposedHelpers.callMethod(card, "getEid", new Object[0]);
            if (eidObj instanceof String && !((String) eidObj).isEmpty()) {
                return (String) eidObj;
            }
        } catch (Throwable ignored) {
        }
        return null;
    }

    private static String eidFromTelephonyManager(Object slot) {
        try {
            Object ctxObj = XposedHelpers.getObjectField(slot, "mContext");
            if (!(ctxObj instanceof android.content.Context)) return null;
            android.content.Context ctx = (android.content.Context) ctxObj;
            Object tm = ctx.getSystemService("phone");
            if (tm == null) return null;
            Method m = tm.getClass().getMethod("getEid", int.class);
            Object r = m.invoke(tm, TARGET_SLOT);
            if (r instanceof String && !((String) r).isEmpty()) {
                return (String) r;
            }
            Method m2 = tm.getClass().getMethod("getEid");
            Object r2 = m2.invoke(tm);
            if (r2 instanceof String && !((String) r2).isEmpty()) {
                return (String) r2;
            }
        } catch (Throwable t) {
            log("TelephonyManager.getEid err: " + t);
        }
        return null;
    }

    private static String eidFromPort(Object slot) {
        try {
            Object card = XposedHelpers.callMethod(slot, "getUiccCard", new Object[0]);
            if (card == null) return null;
            int portCount = 1;
            try {
                Object[] ports = (Object[]) XposedHelpers.callMethod(card, "getUiccPorts",
                        new Object[0]);
                if (ports != null) portCount = ports.length;
            } catch (Throwable ignored) {
            }
            for (int p = 0; p < portCount; p++) {
                try {
                    Object port = XposedHelpers.callMethod(card, "getUiccPort",
                            new Object[]{Integer.valueOf(p)});
                    if (port == null) continue;
                    Object eidObj = XposedHelpers.callMethod(port, "getEid", new Object[0]);
                    if (eidObj instanceof String && !((String) eidObj).isEmpty()) {
                return (String) eidObj;
                    }
                } catch (Throwable ignored) {
                }
            }
        } catch (Throwable ignored) {
        }
        return null;
    }

    private static String loadEid(Object slot) {
        if (sEid != null) return sEid;

        String v = eidFromEuiccCard(slot);
        if (v == null) v = eidFromPort(slot);
        if (v == null) v = eidFromTelephonyManager(slot);

        if (v != null) {
            sEid = v;
            log("EID = " + mask(v) + " (自动获取)");
        } else {
            log("EID 自动获取失败：所有来源为空");
        }
        return sEid;
    }

    private static Set<XC_MethodHook.Unhook> hookAllMethods(
            Class<?> hookClass, String methodName, XC_MethodHook callback) {
        Set<XC_MethodHook.Unhook> hooks = new java.util.HashSet<XC_MethodHook.Unhook>();
        if (hookClass == null) return hooks;
        try {
            for (Method m : hookClass.getDeclaredMethods()) {
                if (m.getName().equals(methodName)) {
                    m.setAccessible(true);
                    hooks.add(XposedBridge.hookMethod(m, callback));
                }
            }
        } catch (Throwable t) {
            log("hookAllMethods error for " + methodName + ": " + t);
        }
        return hooks;
    }

    private static boolean isTargetSlot(XC_MethodHook.MethodHookParam param) {
        try {
            Method m = (Method) param.method;
            Class<?>[] types = m.getParameterTypes();
            for (int i = types.length - 1; i >= 0; i--) {
                if (types[i] == Integer.TYPE) {
                    Object v = (i < param.args.length) ? param.args[i] : null;
                    return (v instanceof Integer) && ((Integer) v) == TARGET_SLOT;
                }
            }
        } catch (Throwable ignored) {
        }
        return false;
    }

    private static void ensureTargetInEuiccSlots(Object controller) {
        try {
            int[] slots = (int[]) XposedHelpers.getObjectField(controller, "mEuiccSlots");
            if (slots == null) {
                XposedHelpers.setObjectField(controller, "mEuiccSlots", new int[]{TARGET_SLOT});
                return;
            }
            for (int s : slots) {
                if (s == TARGET_SLOT) return;
            }
            int[] bigger = new int[slots.length + 1];
            System.arraycopy(slots, 0, bigger, 0, slots.length);
            bigger[slots.length] = TARGET_SLOT;
            XposedHelpers.setObjectField(controller, "mEuiccSlots", bigger);
                    } catch (Throwable t) {
            log("ensureTargetInEuiccSlots err: " + t);
        }
    }

    private static void markSlotAsEuicc(Object slot) {
        try {
            XposedHelpers.setObjectField(slot, "mIsEuicc", Boolean.TRUE);
            XposedHelpers.setObjectField(slot, "mIsRemovable", Boolean.TRUE);
            XposedHelpers.setObjectField(slot, "mActive", Boolean.TRUE);
            String eid = loadEid(slot);
            if (eid != null) {
                XposedHelpers.setObjectField(slot, "mEid", eid);
                log("slot " + TARGET_SLOT + " 已标记为 eUICC，mEid=" + mask(eid));
            } else {
                log("slot " + TARGET_SLOT + " 已标记为 eUICC，EID 待 getEid 时自动获取");
            }
        } catch (Throwable t) {
            log("markSlotAsEuicc err: " + t);
        }
    }

    private static int computeTargetPublicCardId(Object controller) {
        int result = -1;
        try {
            Object slot = XposedHelpers.callMethod(controller, "getUiccSlot",
                    new Object[]{Integer.valueOf(TARGET_SLOT)});
            if (slot == null) return result;

            Object card = XposedHelpers.callMethod(controller, "getUiccCardForSlot",
                    new Object[]{Integer.valueOf(TARGET_SLOT)});
            if (card == null) {
                card = XposedHelpers.callMethod(slot, "getUiccCard", new Object[0]);
            }
            if (card == null) return result;

            Object cardId = XposedHelpers.callMethod(card, "getCardId", new Object[0]);
            if (!(cardId instanceof Integer)) return result;

            String eid = loadEid(slot);
            if (eid == null) return result;

            Object pub = XposedHelpers.callMethod(controller, "convertToPublicCardId",
                    new Object[]{cardId, eid});
            if (pub instanceof Integer) {
                result = ((Integer) pub).intValue();
            } else if (pub instanceof String) {
                result = Integer.parseInt((String) pub);
            }
                    } catch (Throwable t) {
            log("computeTargetPublicCardId err: " + t);
        }
        return result;
    }

    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) throws Throwable {
        if (!"com.android.phone".equals(lpparam.packageName)) {
            return;
        }
        log("hooking telephony UiccSlot/UiccController in " + lpparam.packageName);

        try {
            final Class<?> uiccSlot =
                    XposedHelpers.findClass(CLASS_UICC_SLOT, lpparam.classLoader);
            final Class<?> uiccCtrl =
                    XposedHelpers.findClass(CLASS_UICC_CONTROLLER, lpparam.classLoader);

            hookAllMethods(uiccSlot, "update", new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    if (!isTargetSlot(param)) {
                return;
                    }
                    markSlotAsEuicc(param.thisObject);
                                    }
            });

            hookAllMethods(uiccSlot, "getEid", new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    Object cur = XposedHelpers.getObjectField(param.thisObject, "mEid");
                    if (cur instanceof String && !((String) cur).isEmpty()) {
                return;
                    }
                    String eid = loadEid(param.thisObject);
                    if (eid != null) {
                        XposedHelpers.setObjectField(param.thisObject, "mEid", eid);
            log("getEid 时自动填入 mEid");
                        param.setResult(eid);
                    }
                }
            });

            hookAllMethods(uiccCtrl, "isBuiltInEuiccSlot", new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    Object slotIdx = (param.args.length > 0) ? param.args[0] : null;
                    if (slotIdx instanceof Integer && ((Integer) slotIdx) == TARGET_SLOT) {
                        param.setResult(Boolean.TRUE);
                    }
                }
            });

            hookAllMethods(uiccCtrl, "isRemovableEsimDefaultEuicc", new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    param.setResult(Boolean.TRUE);
                }
            });

            hookAllMethods(uiccCtrl, "setRemovableEsimAsDefaultEuicc", new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    if (param.args.length > 0 && param.args[0] instanceof Boolean) {
                        param.args[0] = Boolean.TRUE;
                    }
                }
            });

            hookAllMethods(uiccCtrl, "onGetSlotStatusDone", new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    ensureTargetInEuiccSlots(param.thisObject);
                }
            });

            hookAllMethods(uiccCtrl, "slotStatusChanged", new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    ensureTargetInEuiccSlots(param.thisObject);
                }
            });

            hookAllMethods(uiccCtrl, "getInstance", new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    try {
                        Object c = param.getResult();
                        XposedHelpers.setObjectField(c, "mHasBuiltInEuicc", Boolean.TRUE);
                        XposedHelpers.setObjectField(c, "mUseRemovableEsimAsDefault", Boolean.TRUE);
                        ensureTargetInEuiccSlots(c);
                    } catch (Throwable ignored) {
                    }
                }
            });

            hookAllMethods(uiccCtrl, "getCardIdForDefaultEuicc", new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    try {
                        int id = computeTargetPublicCardId(param.thisObject);
                        if (id >= 0) {
                            param.setResult(Integer.valueOf(id));
                                                    }
                    } catch (Throwable t) {
            log("getCardIdForDefaultEuicc hook err: " + t);
                    }
                }
            });

            log("UiccSlot + UiccController hooks installed.");
        } catch (Throwable t) {
            log("failed to install hooks: " + t);
        }
    }
}
