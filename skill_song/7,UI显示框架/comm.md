1，页面调用公共接口
#include "mx_user_header.h"
#include "ui_interface.h"
#include "ui_favoriteslabels.h"
#include "ui_withintime.h"
#include "ui_seting.h"
#include "ui_custom.h"
#include "ui_adddetail.h"
#include <stdio.h>
#include <string.h>
/* ========== Display Update Function ========== */
/**
 * Refresh screen based on current state
 * @param ctx Pointer to UI context structure
 * @return true if update successful, false if context is NULL
 */
bool ui_update_display(ui_context_t *ctx)
{
    if (ctx == NULL) return false ;

    printf("UI render: state=%d indicate=%u sub=%u selected=%u\r\n",
           ctx->current_state, ctx->indicate, ctx->sub_indicate, ctx->selected_item);
    
    switch (ctx->current_state) {
        case UI_STATE_MAIN_MENU:  // Main menu state
            ui_draw_main_menu(ctx);
            break;
        case UI_STATE_LABEL_PREVIEW: // Label preview state
            ui_draw_label_preview(ctx);
            break;
        case UI_STATE_DATE: // Label preview state
            ui_draw_date(ctx);
        break;
        case UI_STATE_ADDITEM: // Label preview state
            ui_draw_additem(ctx);
        break;   
        case UI_STATE_CUSTOM: // Custom text state
            ui_draw_custom(ctx);
        break;
        case UI_STATE_BROWSEBYCATEGORY: // Browse by category state
            ui_draw_browsebycategory(ctx);
        break;
        case UI_STATE_FAVORITES: // Favorites state
            ui_draw_favoritesitem(ctx);
        break;
        case UI_STATE_FAVORITES_LABELS: // Favorites Labels state
            ui_draw_favoriteslabels(ctx);
        break;
        case UI_STATE_EXPIRATION_LABEL: // Expiration Label state
            ui_draw_expirationlabel(ctx);
        break;
        case UI_STATE_WITHIN_TIME: // Within time state
            ui_draw_withintime(ctx);
        break;
        case UI_STATE_SETING: // Setting state
            ui_draw_seting(ctx);
        break;
        case UI_STATE_ADDDETAIL: // Add detail state
            ui_draw_adddetail(ctx);
        break;
        default:
            break;
    }
    
    return true;
}
二：公共接口.h
/******************* UI接口头文件 *******************/
#ifndef __ui_interface_H__
#define __ui_interface_H__

#include <stdint.h>
#include <stdbool.h>
//////////////////////////////预览页面值////////////////////////////////////////////////////////
typedef struct {
    uint8_t position;            // 显示位置位掩码：bit0-第1行，bit1-第2行，bit2-第3行，bit3-第4行；或其他视图状态
    char l_type[25];             // 标签类型/日期（示例："COOKED 01 JUN 2025"）
    char l_product_name[25];     // 可选：商品名称（示例："Chicken"），为空字符串表示未设置
    char l_details[25];          // 可选：详细信息（示例："Spicy, contains nuts"），为空字符串表示未设置
    char l_usage_tip[25];        // 可选：使用提示（示例："Use within 3 days"），为空字符串表示未设置
    uint8_t l_print_page;             //打印纸，最小1张最多打印10张
} ui_preview_data_t;
//////////////////////////////时间选择页面////////////////////////////////////////////////////////
typedef struct {
    uint8_t position;            // 显示位置位掩码：bit0-第1行，bit1-第2行，bit2-第3行，bit3-第4行；或其他视图状态
    char l_name[25];             // 标签类型/日期（示例："COOKED 01 JUN 2025"）
    char l_today[25];            // 显示今天日期
    char l_yesterday[25];        // 显示昨天日期
    char l_tomorrow[25];         // 显示明天日期
} ui_date_data_t;
//////////////////////////////add item 值////////////////////////////////////////////////////////
typedef struct {
    uint8_t position;            // 显示位置位掩码：bit0-第1行，bit1-第2行，bit2-第3行，bit3-第4行；用于粉丝框的显示位置
    char l_data;                 //显示单个字符
    bool l_coustom_state;     // 当这个状态为1时，coustom有效
    char l_search_name[25];      // 显示搜索名称，收藏夹共用
    char l_list_1[25];           // 显示列表1
    char l_list_2[25];           // 显示列表1
    char l_list_3[25];           // 显示列表1
    char l_list_4[25];           // 显示列表1
    uint8_t keyboard_mode;
    uint8_t keyboard_index;
    char keyboard_last;
    uint16_t list_offset;
    uint16_t list_total;
    uint8_t list_selected;
} ui_additem_data_t;

//////////////////////////////Custom text page////////////////////////////////////////////////////////
typedef struct {
    char text[25];               // User input text
    char l_list_1[25];           // Suggestion list 1
    char l_list_2[25];           // Suggestion list 2
    char l_list_3[25];           // Suggestion list 3
    char l_list_4[25];           // Suggestion list 4
    uint8_t keyboard_mode;       // 0=letters, 1=numbers/symbols
    uint8_t keyboard_index;      // highlighted key index
    char keyboard_last;          // last rendered key for highlight clearing
    char l_data;                 // current key code/char to render
    uint8_t position;            // 0=no box, 1=draw key box
    uint16_t list_offset;        // paging offset
    uint16_t list_total;         // total match count
    uint8_t list_selected;       // selected row within current page [0..3]
    uint8_t fav_mask;
    bool add_to_favorites;       // save current custom label as favorite label
} ui_custom_data_t;
//////////////////////////////Browse by category 值////////////////////////////////////////////////////////
typedef struct {
    uint8_t position;            // 显示位置位掩码：bit0-第1行，bit1-第2行，bit2-第3行，bit3-第4行；用于粉丝框的显示位置
    char l_list_1[25];           // 显示列表1
    char l_list_2[25];           // 显示列表1
    char l_list_3[25];           // 显示列表1
    char l_list_4[25];           // 显示列表1
    char l_list_5[25];           // 显示列表1
    uint16_t list_total;
    uint16_t wheel_index;
    uint8_t category_group;
    char selected_name[25];
    uint16_t details_offset;
    uint16_t details_total;
    char details_key[5];
} ui_browsebycategory_data_t;
//////////////////////////////Favorites 值////////////////////////////////////////////////////////
typedef struct {
    uint8_t position;            // 显示位置位掩码
    char l_list_1[25];           // 显示列表1
    char l_list_2[25];           // 显示列表2
    char l_list_3[25];           // 显示列表3
    char l_list_4[25];           // 显示列表4
    char l_list_5[25];           // 显示列表5
} ui_favorites_data_t;

//////////////////////////////Favorites Labels 值////////////////////////////////////////////////////////
typedef struct {
    char l_list_1_type[25];      // 列表1类型 (e.g. COOKED)
    char l_list_1_name[25];      // 列表1名称 (e.g. Chicken)
    char l_list_2_type[25];
    char l_list_2_name[25];
    char l_list_3_type[25];
    char l_list_3_name[25];
    char l_list_4_type[25];
    char l_list_4_name[25];
    char l_list_5_type[25];
    char l_list_5_name[25];
} ui_favoriteslabels_data_t;
//////////////////////////////Expiration Label 值////////////////////////////////////////////////////////
typedef struct {
    uint8_t position;            // 显示位置位掩码
    uint16_t year;
    uint8_t month;
    uint8_t day;
    uint8_t nday_idx;
    char l_list_day[4];
    char l_list_month[4];
    char l_list_year[6];
    char l_list_nday[16];
} ui_expirationlabel_data_t;
//////////////////////////////within time////////////////////////////////////////////////////////
typedef struct {
    uint8_t position;            // 显示位置位掩码
    char *l_list_name;            // 列表显示名称
    char *l_list_day;              //显示有效期
    char *l_list_set_data;          // 显示设置天数
    char *l_list_set_date;        // 显示设置日期 天，周，年
} ui_withintime_data_t;
//////////////////////////////////seting/////////////////////////////////////////////////////////////////
typedef struct {
    uint8_t position;               // 显示位置位掩码
    char *l_date_set_day;           // 显示设置天
    char *l_date_set_month;         // 显示设置月
    char *l_date_set_year;          // 显示设置日期年
    char *l_time_set_hour;          // 显示设置时
    char *l_time_set_minute;        // 显示设置分
    char *l_time_set_ampm;          // 显示设置AMPM
    char *l_dateformat_list1;       // 日期格式列表1
    char *l_dateformat_list2;       // 日期格式列表2
    char *l_timeformat_list3;       // 时间格式列表3
    char *l_timeformat_list4;       // 时间格式列表4
    char *l_dateformat_list5;       // 日期格式列表5
    char *l_bat_data;               // 电池电量显示
} ui_seting_data_t;

#define UI_SETING_HAS_L_BAT_DATA 1
////////////////公共页面///////////
/* ========== UI状态定义 ========== */
typedef enum {
    UI_STATE_IDLE = 0,           // 空闲状态
    UI_STATE_MAIN_MENU,          // 主菜单
    UI_STATE_LABEL_PREVIEW,      // 标签预览渲染
    UI_STATE_DATE,               // 日期设置
    UI_STATE_ADDITEM,            // 增加/搜索页面
    UI_STATE_CUSTOM,             // 自定义文本页面
    UI_STATE_BROWSEBYCATEGORY,   // 浏览分类页面
    UI_STATE_FAVORITES,          // 收藏夹页面
    UI_STATE_FAVORITES_LABELS,   // 收藏夹标签页面
    UI_STATE_EXPIRATION_LABEL,   // 过期标签页面
    UI_STATE_WITHIN_TIME,        // 有效期内页面
    UI_STATE_SETING,             // 设置页面
    UI_STATE_ADDDETAIL,          // 增加详情页面
} ui_state_t;
/* ========== 指示值定义 ========== */
typedef enum {
    ui_indicate_off = 0,          // UI显示关闭
    ui_indicate_init = 1,         // UI初始化显示
    ui_indicate_select = 2,       // UI选择显示
} ui_indicate_t;
/* ========== 指示值定义 ========== */
typedef enum {
    ui_sbuindicate_1 = 1,          // 子页面——1
    ui_sbuindicate_2 = 2,          // 子页面——2
    ui_sbuindicate_3 = 3,          // 子页面——3
    ui_sbuindicate_4 = 4,          // 子页面——4
    ui_sbuindicate_5 = 5,          // 子页面——5   
} ui_subindicate_t;
/* ========== UI状态定义 ========== */
typedef enum {
    ui_selectditem_0 = 0,          // 页面显示局部组件
    ui_selectditem_1 = 1,          // 页面显示局部组件
    ui_selectditem_2 = 2,          // 页面显示局部组件
    ui_selectditem_3 = 3,          // 页面显示局部组件
    ui_selectditem_4 = 4,          // 页面显示局部组件
    ui_selectditem_5 = 5,          // 页面显示局部组件
    ui_selectditem_6 = 6,          // 页面显示局部组件
    ui_selectditem_7 = 7,          // 页面显示局部组件 
    ui_selectditem_8 = 8,          // 页面显示局部组件
    ui_selectditem_9 = 9,          // 页面显示局部组件
    ui_selectditem_10 = 10,        // 页面显示局部组件    
} ui_selectditem_t;

/** UI全局上下文数据结构 */
typedef struct {
    
    /* 页面状态数据 */
    uint8_t indicate;            // 页面显示状态（0=不显示，1=初始化页面，2=选择/动态页面；常与 selected_item 配合使用）
    uint8_t sub_indicate;        // 子页面显示状态；1 表示单个子页面；sub_indicate=N 表示 N 个子页面
    uint8_t selected_item;       // 选中的元素：页面上哪个 UI 区块需要更新 上下选择
    uint16_t selected_sub_item;   // 选中的子元素：指向哪个子页面进行渲染 左右选择
    ui_state_t current_state;    // 当前 UI 状态：指向要渲染的页面
    ui_preview_data_t preview_data;      // 传递到预览页面的数据载荷
    ui_date_data_t date_data;            // 传递到日期页面的数据载荷
    ui_additem_data_t additem_data;      // 传递到增加/搜索页面的数据载荷
    ui_custom_data_t custom_data;        // 传递到自定义文本页面的数据载荷
    ui_browsebycategory_data_t browsebycategory_data;      // 传递到浏览分类页面的数据载荷
    ui_favorites_data_t favorites_data;      // 传递到收藏夹页面的数据载荷
    ui_favoriteslabels_data_t favoriteslabels_data; // 传递到收藏夹标签页面的数据载荷
    ui_expirationlabel_data_t expirationlabel_data; // 传递到过期标签页面的数据载荷
    ui_withintime_data_t withintime_data; // 传递到有效期内页面的数据载荷
    ui_seting_data_t seting_data; // 传递到设置页面的数据载荷
    uint8_t favorites_return_state;
} ui_context_t;


/////////////////////////////////// 全局接口函数 ////////////////////////////////
/* ========== 显示更新函数 ========== */
/**
 * 根据当前状态刷新屏幕
 * @param ctx UI上下文结构体指针
 * @return 成功返回 true；ctx 为 NULL 返回 false
 */
bool ui_update_display(ui_context_t *ctx);
#endif /*__ui_interface_H__*/


