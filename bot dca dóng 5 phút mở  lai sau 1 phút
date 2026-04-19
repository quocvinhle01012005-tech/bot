//+------------------------------------------------------------------+
//| SimpleGrid.mq5                                                   |
//| DCA Grid - tu dong dong sau 5 phut, mo lai sau 1 phut            |
//+------------------------------------------------------------------+
#property copyright "SimpleGrid"
#property version   "1.04"

#include <Trade/Trade.mqh>

enum ENUM_TRADING_DIRECTION
{
   DIRECTION_BOTH      = 0,
   DIRECTION_BUY_ONLY  = 1,
   DIRECTION_SELL_ONLY = 2
};

input group "=== TRADING CONFIGURATION ==="
input ENUM_TRADING_DIRECTION TradingDirection  = DIRECTION_BOTH;
input double   InitialLotSize    = 0.01;
input int      DCA_Distance      = 7;
input double   DCA_Multiplier    = 1.3;
input int      Max_Grid_Levels   = 22;
input int      CycleOpenMinutes  = 5;
input int      CycleWaitMinutes  = 1;

input group "=== SYSTEM ==="
input int      BuyMagicNumber  = 666666;
input int      SellMagicNumber = 888888;
input int      DeviationPoints = 30;

input group "=== CHART BUTTONS ==="
input bool     ButtonCornerTop = true;
input int      ButtonOffsetX   = 12;
input int      ButtonOffsetY   = 28;
input int      ButtonWidth     = 118;
input int      ButtonHeight    = 24;
input int      ButtonGap       = 4;

struct DCALevel
{
   double price;
   double lot_size;
   ulong  ticket;
   bool   is_filled;
};

DCALevel buy_grid[];
DCALevel sell_grid[];
int      active_buy_levels   = 0;
int      active_sell_levels  = 0;
bool     buy_grid_active     = false;
bool     sell_grid_active    = false;

datetime g_cycle_open_time   = 0;
datetime g_cycle_close_time  = 0;
bool     g_waiting_to_reopen = false;

CTrade   trade;
int      g_pip_points = 10;

const string BTN_START_BUY  = "SimpleGrid_StartBuy";
const string BTN_START_SELL = "SimpleGrid_StartSell";
const string BTN_CLOSE_BUY  = "SimpleGrid_CloseBuy";
const string BTN_CLOSE_SELL = "SimpleGrid_CloseSell";
const string BTN_CLOSE_ALL  = "SimpleGrid_CloseAll";

//+------------------------------------------------------------------+
int OnInit()
{
   int digits = (int)SymbolInfoInteger(_Symbol, SYMBOL_DIGITS);
   g_pip_points = (digits == 2 || digits == 4) ? 10 : 100;

   ArrayResize(buy_grid,  Max_Grid_Levels);
   ArrayResize(sell_grid, Max_Grid_Levels);

   trade.SetDeviationInPoints(DeviationPoints);
   trade.SetTypeFillingBySymbol(_Symbol);

   CreateOrUpdateButtons();
   return INIT_SUCCEEDED;
}

//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
   ObjectDelete(0, BTN_START_BUY);
   ObjectDelete(0, BTN_START_SELL);
   ObjectDelete(0, BTN_CLOSE_BUY);
   ObjectDelete(0, BTN_CLOSE_SELL);
   ObjectDelete(0, BTN_CLOSE_ALL);
   ChartRedraw();
}

//+------------------------------------------------------------------+
void OnTick()
{
   ProcessCycleTimer();
   ProcessGridLoop();
}

//+------------------------------------------------------------------+
void ProcessCycleTimer()
{
   datetime now = TimeCurrent();

   if(g_waiting_to_reopen)
   {
      int waited = (int)(now - g_cycle_close_time);
      if(waited >= CycleWaitMinutes * 60)
      {
         Print("SimpleGrid: het cho ", CycleWaitMinutes, " phut - mo lai luoi.");
         g_waiting_to_reopen = false;
         AutoStartGrids();
      }
      return;
   }

   if(buy_grid_active || sell_grid_active)
   {
      int elapsed = (int)(now - g_cycle_open_time);
      if(elapsed >= CycleOpenMinutes * 60)
      {
         Print("SimpleGrid: het ", CycleOpenMinutes, " phut - dong tat ca lenh.");
         CloseAllBuyPositions();
         CloseAllSellPositions();
         g_cycle_close_time  = now;
         g_waiting_to_reopen = true;
      }
   }
}

//+------------------------------------------------------------------+
void AutoStartGrids()
{
   if(TradingDirection != DIRECTION_SELL_ONLY)
      StartBuyDCAGrid();
   if(TradingDirection != DIRECTION_BUY_ONLY)
      StartSellDCAGrid();
}

//+------------------------------------------------------------------+
void ProcessGridLoop()
{
   if(g_waiting_to_reopen) return;

   ValidateGridState();

   if(buy_grid_active)  ManageBuyDCAGrid();
   if(sell_grid_active) ManageSellDCAGrid();

   UpdateButtonAppearance();
}

//+------------------------------------------------------------------+
double PipSizePrice()
{
   return _Point * (double)g_pip_points;
}

//+------------------------------------------------------------------+
double NormalizeLotSize(double lot_size)
{
   double min_lot  = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
   double max_lot  = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);
   double lot_step = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);

   if(lot_size < min_lot) lot_size = min_lot;
   if(lot_size > max_lot) lot_size = max_lot;

   lot_size = NormalizeDouble(MathRound(lot_size / lot_step) * lot_step, 2);
   return lot_size;
}

//+------------------------------------------------------------------+
void ResetGrid(DCALevel &grid[])
{
   for(int k = 0; k < Max_Grid_Levels; k++)
   {
      grid[k].price     = 0;
      grid[k].lot_size  = 0;
      grid[k].ticket    = 0;
      grid[k].is_filled = false;
   }
}

//+------------------------------------------------------------------+
void ValidateGridState()
{
   int actual_buy = 0, actual_sell = 0;

   for(int i = 0; i < PositionsTotal(); i++)
   {
      ulong ticket = PositionGetTicket(i);
      if(ticket == 0) continue;
      if(PositionGetString(POSITION_SYMBOL) != _Symbol) continue;
      long magic = PositionGetInteger(POSITION_MAGIC);
      long typ   = PositionGetInteger(POSITION_TYPE);
      if(magic == BuyMagicNumber  && typ == POSITION_TYPE_BUY)  actual_buy++;
      if(magic == SellMagicNumber && typ == POSITION_TYPE_SELL) actual_sell++;
   }

   if(buy_grid_active && actual_buy == 0)
   {
      buy_grid_active   = false;
      active_buy_levels = 0;
      ResetGrid(buy_grid);
   }
   if(sell_grid_active && actual_sell == 0)
   {
      sell_grid_active   = false;
      active_sell_levels = 0;
      ResetGrid(sell_grid);
   }

   if(buy_grid_active  && active_buy_levels  != actual_buy)  active_buy_levels  = actual_buy;
   if(sell_grid_active && active_sell_levels != actual_sell) active_sell_levels = actual_sell;
}

//+------------------------------------------------------------------+
void RequestStartBuyGrid()
{
   if(TradingDirection == DIRECTION_SELL_ONLY) { Print("SimpleGrid: buy disabled."); return; }
   if(buy_grid_active) { Print("SimpleGrid: buy grid already active."); return; }
   StartBuyDCAGrid();
}

//+------------------------------------------------------------------+
void RequestStartSellGrid()
{
   if(TradingDirection == DIRECTION_BUY_ONLY) { Print("SimpleGrid: sell disabled."); return; }
   if(sell_grid_active) { Print("SimpleGrid: sell grid already active."); return; }
   StartSellDCAGrid();
}

//+------------------------------------------------------------------+
void StartBuyDCAGrid()
{
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   double pip = PipSizePrice();

   buy_grid_active   = true;
   active_buy_levels = 0;
   g_cycle_open_time = TimeCurrent();

   double current_lot = InitialLotSize;
   for(int i = 0; i < Max_Grid_Levels; i++)
   {
      buy_grid[i].price     = ask - (i * DCA_Distance * pip);
      current_lot           = NormalizeLotSize(current_lot);
      buy_grid[i].lot_size  = current_lot;
      buy_grid[i].ticket    = 0;
      buy_grid[i].is_filled = false;
      current_lot = NormalizeDouble(current_lot * DCA_Multiplier, 2);
   }

   OpenBuyLevel(0);
   Print("SimpleGrid: buy grid started @ ", ask);
}

//+------------------------------------------------------------------+
void OpenBuyLevel(const int level)
{
   MqlTradeRequest request = {};
   MqlTradeResult  result  = {};

   double ask      = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   double lot_size = NormalizeLotSize(buy_grid[level].lot_size);

   request.action  = TRADE_ACTION_DEAL;
   request.symbol  = _Symbol;
   request.volume  = lot_size;
   request.type    = ORDER_TYPE_BUY;
   request.price   = ask;
   request.magic   = BuyMagicNumber;
   request.comment = "SimpleGrid Buy L" + IntegerToString(level);

   if(!OrderSend(request, result)) { Print("SimpleGrid: OpenBuyLevel error ", GetLastError()); return; }
   if(result.retcode != TRADE_RETCODE_DONE && result.retcode != TRADE_RETCODE_PLACED)
   { Print("SimpleGrid: OpenBuyLevel retcode ", result.retcode); return; }

   buy_grid[level].ticket    = result.order;
   buy_grid[level].is_filled = true;
   buy_grid[level].lot_size  = lot_size;
   active_buy_levels++;
}

//+------------------------------------------------------------------+
void ManageBuyDCAGrid()
{
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   for(int i = 1; i < Max_Grid_Levels; i++)
   {
      if(!buy_grid[i].is_filled && bid <= buy_grid[i].price)
      {
         OpenBuyLevel(i);
         break;
      }
   }
}

//+------------------------------------------------------------------+
void StartSellDCAGrid()
{
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double pip = PipSizePrice();

   sell_grid_active   = true;
   active_sell_levels = 0;
   g_cycle_open_time  = TimeCurrent();

   double current_lot = InitialLotSize;
   for(int i = 0; i < Max_Grid_Levels; i++)
   {
      sell_grid[i].price     = bid + (i * DCA_Distance * pip);
      current_lot            = NormalizeLotSize(current_lot);
      sell_grid[i].lot_size  = current_lot;
      sell_grid[i].ticket    = 0;
      sell_grid[i].is_filled = false;
      current_lot = NormalizeDouble(current_lot * DCA_Multiplier, 2);
   }

   OpenSellLevel(0);
   Print("SimpleGrid: sell grid started @ ", bid);
}

//+------------------------------------------------------------------+
void OpenSellLevel(const int level)
{
   MqlTradeRequest request = {};
   MqlTradeResult  result  = {};

   double bid      = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   double lot_size = NormalizeLotSize(sell_grid[level].lot_size);

   request.action  = TRADE_ACTION_DEAL;
   request.symbol  = _Symbol;
   request.volume  = lot_size;
   request.type    = ORDER_TYPE_SELL;
   request.price   = bid;
   request.magic   = SellMagicNumber;
   request.comment = "SimpleGrid Sell L" + IntegerToString(level);

   if(!OrderSend(request, result)) { Print("SimpleGrid: OpenSellLevel error ", GetLastError()); return; }
   if(result.retcode != TRADE_RETCODE_DONE && result.retcode != TRADE_RETCODE_PLACED)
   { Print("SimpleGrid: OpenSellLevel retcode ", result.retcode); return; }

   sell_grid[level].ticket    = result.order;
   sell_grid[level].is_filled = true;
   sell_grid[level].lot_size  = lot_size;
   active_sell_levels++;
}

//+------------------------------------------------------------------+
void ManageSellDCAGrid()
{
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   for(int i = 1; i < Max_Grid_Levels; i++)
   {
      if(!sell_grid[i].is_filled && ask >= sell_grid[i].price)
      {
         OpenSellLevel(i);
         break;
      }
   }
}

//+------------------------------------------------------------------+
void CloseAllBuyPositions()
{
   ulong tickets[];
   int n = 0;
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      ulong t = PositionGetTicket(i);
      if(t == 0) continue;
      if(PositionGetString(POSITION_SYMBOL) != _Symbol) continue;
      if(PositionGetInteger(POSITION_MAGIC) != BuyMagicNumber) continue;
      if(PositionGetInteger(POSITION_TYPE)  != POSITION_TYPE_BUY) continue;
      ArrayResize(tickets, n + 1);
      tickets[n++] = t;
   }
   for(int j = 0; j < n; j++)
      if(!trade.PositionClose(tickets[j]))
         Print("SimpleGrid: close buy #", tickets[j], " failed");

   buy_grid_active   = false;
   active_buy_levels = 0;
   ResetGrid(buy_grid);
}

//+------------------------------------------------------------------+
void CloseAllSellPositions()
{
   ulong tickets[];
   int n = 0;
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      ulong t = PositionGetTicket(i);
      if(t == 0) continue;
      if(PositionGetString(POSITION_SYMBOL) != _Symbol) continue;
      if(PositionGetInteger(POSITION_MAGIC) != SellMagicNumber) continue;
      if(PositionGetInteger(POSITION_TYPE)  != POSITION_TYPE_SELL) continue;
      ArrayResize(tickets, n + 1);
      tickets[n++] = t;
   }
   for(int j = 0; j < n; j++)
      if(!trade.PositionClose(tickets[j]))
         Print("SimpleGrid: close sell #", tickets[j], " failed");

   sell_grid_active   = false;
   active_sell_levels = 0;
   ResetGrid(sell_grid);
}

//+------------------------------------------------------------------+
void CreateOrUpdateButtons()
{
   ENUM_BASE_CORNER corner = ButtonCornerTop ? CORNER_LEFT_UPPER : CORNER_LEFT_LOWER;
   int y = ButtonOffsetY;
   int x = ButtonOffsetX;
   int h = ButtonHeight + ButtonGap;

   CreateButton(BTN_START_BUY,  "Start BUY",  x, y, corner); y += h;
   CreateButton(BTN_CLOSE_BUY,  "Close BUY",  x, y, corner); y += h;
   CreateButton(BTN_START_SELL, "Start SELL", x, y, corner); y += h;
   CreateButton(BTN_CLOSE_SELL, "Close SELL", x, y, corner); y += h;
   CreateButton(BTN_CLOSE_ALL,  "Close ALL",  x, y, corner);

   UpdateButtonAppearance();
}

//+------------------------------------------------------------------+
void CreateButton(const string name, const string text, const int xdist, const int ydist,
                  const ENUM_BASE_CORNER corner)
{
   if(ObjectFind(0, name) >= 0) { ObjectSetString(0, name, OBJPROP_TEXT, text); return; }
   if(!ObjectCreate(0, name, OBJ_BUTTON, 0, 0, 0)) return;
   ObjectSetInteger(0, name, OBJPROP_CORNER,       corner);
   ObjectSetInteger(0, name, OBJPROP_XDISTANCE,    xdist);
   ObjectSetInteger(0, name, OBJPROP_YDISTANCE,    ydist);
   ObjectSetInteger(0, name, OBJPROP_XSIZE,        ButtonWidth);
   ObjectSetInteger(0, name, OBJPROP_YSIZE,        ButtonHeight);
   ObjectSetString(0,  name, OBJPROP_TEXT,         text);
   ObjectSetInteger(0, name, OBJPROP_COLOR,        clrWhite);
   ObjectSetInteger(0, name, OBJPROP_BGCOLOR,      clrDimGray);
   ObjectSetInteger(0, name, OBJPROP_BORDER_COLOR, clrSilver);
   ObjectSetInteger(0, name, OBJPROP_SELECTABLE,   false);
}

//+------------------------------------------------------------------+
void UpdateButtonAppearance()
{
   bool allow_buy  = (TradingDirection != DIRECTION_SELL_ONLY);
   bool allow_sell = (TradingDirection != DIRECTION_BUY_ONLY);

   color buy_bg  = clrGreen;
   color sell_bg = clrFireBrick;

   if(g_waiting_to_reopen)
   {
      buy_bg  = clrDarkOrange;
      sell_bg = clrDarkOrange;
   }
   else
   {
      if(buy_grid_active)  buy_bg  = clrDarkGreen;
      if(sell_grid_active) sell_bg = clrMaroon;
   }

   if(ObjectFind(0, BTN_START_BUY) >= 0)
   {
      ObjectSetInteger(0, BTN_START_BUY, OBJPROP_BGCOLOR, allow_buy ? buy_bg : clrGray);
      string lbl = buy_grid_active ? "BUY (active)" : (g_waiting_to_reopen ? "BUY (waiting...)" : "Start BUY");
      ObjectSetString(0, BTN_START_BUY, OBJPROP_TEXT, lbl);
   }
   if(ObjectFind(0, BTN_START_SELL) >= 0)
   {
      ObjectSetInteger(0, BTN_START_SELL, OBJPROP_BGCOLOR, allow_sell ? sell_bg : clrGray);
      string lbl = sell_grid_active ? "SELL (active)" : (g_waiting_to_reopen ? "SELL (waiting...)" : "Start SELL");
      ObjectSetString(0, BTN_START_SELL, OBJPROP_TEXT, lbl);
   }
}

//+------------------------------------------------------------------+
void OnChartEvent(const int id, const long &lparam, const double &dparam, const string &sparam)
{
   if(id != CHARTEVENT_OBJECT_CLICK) return;

   if(sparam == BTN_START_BUY)
   {
      ObjectSetInteger(0, BTN_START_BUY, OBJPROP_STATE, false);
      RequestStartBuyGrid();
   }
   else if(sparam == BTN_START_SELL)
   {
      ObjectSetInteger(0, BTN_START_SELL, OBJPROP_STATE, false);
      RequestStartSellGrid();
   }
   else if(sparam == BTN_CLOSE_BUY)
   {
      ObjectSetInteger(0, BTN_CLOSE_BUY, OBJPROP_STATE, false);
      g_waiting_to_reopen = false;
      CloseAllBuyPositions();
   }
   else if(sparam == BTN_CLOSE_SELL)
   {
      ObjectSetInteger(0, BTN_CLOSE_SELL, OBJPROP_STATE, false);
      g_waiting_to_reopen = false;
      CloseAllSellPositions();
   }
   else if(sparam == BTN_CLOSE_ALL)
   {
      ObjectSetInteger(0, BTN_CLOSE_ALL, OBJPROP_STATE, false);
      g_waiting_to_reopen = false;
      CloseAllBuyPositions();
      CloseAllSellPositions();
   }

   UpdateButtonAppearance();
   ChartRedraw();
}
//+------------------------------------------------------------------+
