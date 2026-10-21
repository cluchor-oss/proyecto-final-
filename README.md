# proyecto-final-
El horno de la 2
/ <summary>Objetivo: transportar los productos entre presentación y negocio. Cantidad siempre en la unidad indicada.</summary>
public sealed record Producto(string pStrCodigoCM, string pStrProductoCM, string pStrCategoriaCM,
   string pStrUnidadCM, decimal pDecCantidadCM, decimal pDecPrecioUnidadCM,
   string pStrUbicacionCM, DateTime pDtmCreacionCM, DateTime? pDtmVencimientoCM);
 
/// <summary>Objetivo: identificar al cliente sin confundir NIT con un número entero.</summary>
public sealed record Cliente(string pStrNitCM, string pStrNombreCM, string pStrApellidoCM,
   string pStrCiudadCM, string pStrDireccionCM);
 
/// <summary>Objetivo: representar una línea del carrito; el servidor determina el precio vigente.</summary>
public sealed record LineaVenta(string pStrCodigoCM, decimal pDecCantidadCM);
 
/// <summary>Objetivo: registrar obligaciones, nómina, gastos fijos y pedidos de proveedores.</summary>
public sealed record Cuenta(string pStrTipoCM, string pStrProveedorCM, string pStrProductoCM,
   string pStrPedidoCM, decimal pDecCantidadCM, decimal pDecPrecioUnidadCM,
   DateTime pDtmPedidoCM, DateTime pDtmEntregaCM, string pStrRecibeCM);
 
/// <summary>Objetivo: comparar escenarios con el mismo capital fijo y distinta cantidad de trabajadores.</summary>
public sealed record Escenario(string pStrGrupoCM, string pStrCodigoCM, int pIntTrabajadoresCM,
   decimal pDecProduccionCM, decimal pDecCostoFijoCM, decimal pDecCostoVariableCM,
   decimal pDecPrecioCM, decimal pDecDemandaCM, bool pBlnRealCM, DateTime pDtmFechaCM);
 
LOGICA DE NEGOCIO
using System.Data;
using Microsoft.Data.SqlClient;
using Panaderia.Entidades;
 
namespace Panaderia.LogicaNegocio;
 
/// <summary>Objetivo: centralizar validación, persistencia y operaciones de panadería. La presentación no ejecuta SQL.</summary>
public sealed class Servicio
{
   private readonly string gStrConexionCM;
   /// <summary>Recibe una conexión configurada externamente.</summary>
   public Servicio(string pStrConexionCM) => gStrConexionCM = pStrConexionCM;
 
   /// <summary>Valida las credenciales académicas solicitadas; no es autenticación de SQL Server.</summary>
   public bool Login(string pStrUsuarioCM, string pStrClaveCM) => pStrUsuarioCM == "admin" && pStrClaveCM == "panaderia2026";
 
   /// <summary>Ejecuta consultas parametrizadas y devuelve un conjunto independiente de la conexión.</summary>
   private DataTable Consultar(string pStrSqlCM, params SqlParameter[] pArrParametrosCM)
   {
       using var lObjConexionCM = new SqlConnection(gStrConexionCM);
       using var lObjComandoCM = new SqlCommand(pStrSqlCM, lObjConexionCM);
       lObjComandoCM.Parameters.AddRange(pArrParametrosCM);
       using var lObjAdaptadorCM = new SqlDataAdapter(lObjComandoCM);
       var lDtbResultadoCM = new DataTable();
       lObjAdaptadorCM.Fill(lDtbResultadoCM);
       return lDtbResultadoCM;
   }
 
   /// <summary>Ejecuta una escritura; cada procedimiento controla su transacción cuando afecta varias tablas.</summary>
   private void Ejecutar(string pStrSqlCM, params SqlParameter[] pArrParametrosCM)
   {
       using var lObjConexionCM = new SqlConnection(gStrConexionCM);
       using var lObjComandoCM = new SqlCommand(pStrSqlCM, lObjConexionCM);
       lObjComandoCM.Parameters.AddRange(pArrParametrosCM);
       lObjConexionCM.Open(); lObjComandoCM.ExecuteNonQuery();
   }
 
   /// <summary>Crea parámetros sin concatenar entradas del usuario en SQL.</summary>
   private static SqlParameter Parametro(string pStrNombreCM, object? pObjValorCM) => new(pStrNombreCM, pObjValorCM ?? DBNull.Value);
   /// <summary>Rechaza los campos obligatorios vacíos.</summary>
   private static void Requerido(string pStrValorCM, string pStrCampoCM)
   { if (string.IsNullOrWhiteSpace(pStrValorCM)) throw new ArgumentException($"Ingrese {pStrCampoCM}."); }
 
   /// <summary>Comprueba disponibilidad de la base sin modificar información.</summary>
   public void ProbarConexion() => Consultar("SELECT TOP (0) Codigo FROM dbo.Inventario");
   /// <summary>Lista inventario o únicamente productos terminados disponibles y no vencidos.</summary>
   public DataTable Inventario(bool pBlnVentaCM = false) => Consultar("SELECT * FROM dbo.Inventario" +
       (pBlnVentaCM ? " WHERE Categoria='Terminado' AND Cantidad>0 AND (FechaVencimiento IS NULL OR FechaVencimiento>=CAST(GETDATE() AS date))" : "") + " ORDER BY Producto");
   /// <summary>Actualiza precio y ubicación sin alterar existencias auditadas.</summary>
   public void ActualizarPrecio(string pStrCodigoCM, decimal pDecPrecioCM, string pStrUbicacionCM)
   {
       if (pDecPrecioCM < 0) throw new ArgumentException("Precio negativo.");
       Ejecutar("UPDATE dbo.Inventario SET PrecioUnidad=@precio,Ubicacion=@ubicacion WHERE Codigo=@codigo", Parametro("@codigo", pStrCodigoCM), Parametro("@precio", pDecPrecioCM), Parametro("@ubicacion", pStrUbicacionCM));
   }
   /// <summary>Da de alta un producto. Las entradas posteriores se registran como movimientos, sin sobrescribir el stock.</summary>
   public void CrearProducto(Producto pObjProductoCM)
   {
       Requerido(pObjProductoCM.pStrCodigoCM, "código"); Requerido(pObjProductoCM.pStrProductoCM, "producto");
       if (pObjProductoCM.pDecCantidadCM < 0 || pObjProductoCM.pDecPrecioUnidadCM < 0) throw new ArgumentException("Cantidad y precio deben ser no negativos.");
       if (pObjProductoCM.pDtmVencimientoCM < pObjProductoCM.pDtmCreacionCM) throw new ArgumentException("Vencimiento anterior a la creación.");
       Ejecutar("EXEC dbo.CrearProducto @codigo,@producto,@categoria,@unidad,@cantidad,@precio,@ubicacion,@fecha,@vence",
           Parametro("@codigo", pObjProductoCM.pStrCodigoCM), Parametro("@producto", pObjProductoCM.pStrProductoCM),
           Parametro("@categoria", pObjProductoCM.pStrCategoriaCM), Parametro("@unidad", pObjProductoCM.pStrUnidadCM),
           Parametro("@cantidad", pObjProductoCM.pDecCantidadCM), Parametro("@precio", pObjProductoCM.pDecPrecioUnidadCM),
           Parametro("@ubicacion", pObjProductoCM.pStrUbicacionCM), Parametro("@fecha", pObjProductoCM.pDtmCreacionCM.Date), Parametro("@vence", pObjProductoCM.pDtmVencimientoCM?.Date));
   }
   /// <summary>Registra compras, consumo de ingredientes o merma, manteniendo cantidades no negativas.</summary>
   public void Movimiento(string pStrCodigoCM, decimal pDecCantidadCM, string pStrMotivoCM)
   {
       Requerido(pStrCodigoCM, "código"); Requerido(pStrMotivoCM, "motivo");
       if (pDecCantidadCM == 0) throw new ArgumentException("El movimiento debe ser distinto de cero.");
       Ejecutar("EXEC dbo.MoverStock @codigo,@cantidad,@motivo", Parametro("@codigo", pStrCodigoCM), Parametro("@cantidad", pDecCantidadCM), Parametro("@motivo", pStrMotivoCM));
   }
   /// <summary>Recupera un cliente por NIT.</summary>
   public DataTable BuscarCliente(string pStrNitCM) => Consultar("SELECT * FROM dbo.Clientes WHERE Nit=@nit", Parametro("@nit", pStrNitCM.Trim().ToUpperInvariant()));
   /// <summary>Guarda un cliente nuevo o actualiza su dirección y nombre conservando el historial de ventas.</summary>
   public void GuardarCliente(Cliente pObjClienteCM)
   {
       Requerido(pObjClienteCM.pStrNitCM, "NIT"); Requerido(pObjClienteCM.pStrNombreCM, "nombre"); Requerido(pObjClienteCM.pStrDireccionCM, "dirección");
       Ejecutar("EXEC dbo.GuardarCliente @nit,@nombre,@apellido,@ciudad,@direccion", Parametro("@nit", pObjClienteCM.pStrNitCM.Trim().ToUpperInvariant()), Parametro("@nombre", pObjClienteCM.pStrNombreCM), Parametro("@apellido", pObjClienteCM.pStrApellidoCM), Parametro("@ciudad", pObjClienteCM.pStrCiudadCM), Parametro("@direccion", pObjClienteCM.pStrDireccionCM));
   }
   /// <summary>Envía el carrito como tabla tipada; venta y descuento de stock se confirman juntos o se revierten juntos.</summary>
   public int Vender(string pStrNitCM, string pStrPagoCM, IReadOnlyCollection<LineaVenta> pLstLineasCM)
   {
       if (pLstLineasCM.Count == 0 || pLstLineasCM.Any(pObjLineaCM => pObjLineaCM.pDecCantidadCM <= 0)) throw new ArgumentException("Agregue cantidades positivas al carrito.");
       var lDtbLineasCM = new DataTable(); lDtbLineasCM.Columns.Add("Codigo", typeof(string)); lDtbLineasCM.Columns.Add("Cantidad", typeof(decimal));
       foreach (var lObjLineaCM in pLstLineasCM) lDtbLineasCM.Rows.Add(lObjLineaCM.pStrCodigoCM, lObjLineaCM.pDecCantidadCM);
       var lObjParametroCM = new SqlParameter("@lineas", SqlDbType.Structured) { TypeName = "dbo.LineasVenta", Value = lDtbLineasCM };
       return Convert.ToInt32(Consultar("EXEC dbo.RegistrarVenta @nit,@pago,@lineas", Parametro("@nit", pStrNitCM.Trim().ToUpperInvariant()), Parametro("@pago", pStrPagoCM), lObjParametroCM).Rows[0][0]);
   }
   /// <summary>Lista datos de clientes.</summary>
   public DataTable Clientes() => Consultar("SELECT * FROM dbo.Clientes ORDER BY Nombre");
   /// <summary>Lista compras en la fecha seleccionada, opcionalmente filtradas por NIT.</summary>
   public DataTable Historial(DateTime pDtmFechaCM, string pStrNitCM) => Consultar("SELECT * FROM dbo.Ventas WHERE Fecha>=@fecha AND Fecha<DATEADD(day,1,@fecha) AND (@nit='' OR Nit=@nit) ORDER BY IdVenta", Parametro("@fecha", pDtmFechaCM.Date), Parametro("@nit", pStrNitCM.Trim().ToUpperInvariant()));
   /// <summary>Registra un pedido u obligación; guardar el pedido no implica recibir inventario.</summary>
   public void GuardarCuenta(Cuenta pObjCuentaCM)
   {
       Requerido(pObjCuentaCM.pStrProveedorCM, "proveedor o beneficiario"); Requerido(pObjCuentaCM.pStrProductoCM, "concepto");
       if (pObjCuentaCM.pDecCantidadCM <= 0 || pObjCuentaCM.pDecPrecioUnidadCM < 0 || pObjCuentaCM.pDtmEntregaCM.Date < pObjCuentaCM.pDtmPedidoCM.Date) throw new ArgumentException("Revise cantidades, precios y fechas.");
       Ejecutar("INSERT dbo.CuentasPorPagar(Tipo,Proveedor,Producto,Pedido,Cantidad,PrecioUnidad,FechaPedido,FechaEntrega,QuienRecibe) VALUES(@tipo,@proveedor,@producto,@pedido,@cantidad,@precio,@fecha,@entrega,@recibe)",
           Parametro("@tipo", pObjCuentaCM.pStrTipoCM), Parametro("@proveedor", pObjCuentaCM.pStrProveedorCM), Parametro("@producto", pObjCuentaCM.pStrProductoCM), Parametro("@pedido", pObjCuentaCM.pStrPedidoCM), Parametro("@cantidad", pObjCuentaCM.pDecCantidadCM), Parametro("@precio", pObjCuentaCM.pDecPrecioUnidadCM), Parametro("@fecha", pObjCuentaCM.pDtmPedidoCM.Date), Parametro("@entrega", pObjCuentaCM.pDtmEntregaCM.Date), Parametro("@recibe", pObjCuentaCM.pStrRecibeCM));
   }
   /// <summary>Lista obligaciones y sus saldos.</summary>
   public DataTable Cuentas() => Consultar("SELECT *,PrecioTotal-Pagado AS Saldo FROM dbo.CuentasPorPagar ORDER BY FechaEntrega");
   /// <summary>Abona sin permitir sobrepago ni duplicar una actualización concurrente.</summary>
   public void Abonar(int pIntIdCM, decimal pDecMontoCM) => Ejecutar("EXEC dbo.AbonarCuenta @id,@monto", Parametro("@id", pIntIdCM), Parametro("@monto", pDecMontoCM));
   /// <summary>Registra escenario o producción real; solo la real agrega existencias. Usa una serie comparable por producto y fecha.</summary>
   public void Produccion(Escenario pObjEscenarioCM)
   {
       Requerido(pObjEscenarioCM.pStrGrupoCM, "serie comparable");
       if (pObjEscenarioCM.pIntTrabajadoresCM <= 0 || pObjEscenarioCM.pDecProduccionCM < 0 || pObjEscenarioCM.pDecCostoFijoCM < 0 || pObjEscenarioCM.pDecCostoVariableCM < 0 || pObjEscenarioCM.pDecPrecioCM < 0 || pObjEscenarioCM.pDecDemandaCM < 0) throw new ArgumentException("Trabajadores debe ser positivo; los demás valores no negativos.");
       Ejecutar("EXEC dbo.RegistrarProduccion @grupo,@codigo,@trabajadores,@q,@cf,@cv,@precio,@demanda,@real,@fecha",
           Parametro("@grupo", pObjEscenarioCM.pStrGrupoCM), Parametro("@codigo", pObjEscenarioCM.pStrCodigoCM), Parametro("@trabajadores", pObjEscenarioCM.pIntTrabajadoresCM), Parametro("@q", pObjEscenarioCM.pDecProduccionCM), Parametro("@cf", pObjEscenarioCM.pDecCostoFijoCM), Parametro("@cv", pObjEscenarioCM.pDecCostoVariableCM), Parametro("@precio", pObjEscenarioCM.pDecPrecioCM), Parametro("@demanda", pObjEscenarioCM.pDecDemandaCM), Parametro("@real", pObjEscenarioCM.pBlnRealCM), Parametro("@fecha", pObjEscenarioCM.pDtmFechaCM.Date));
   }
   /// <summary>Devuelve PFT, PFM, PFMg, etapas y equilibrio con supuestos explícitos en la guía.</summary>
   public DataTable Proyecciones(DateTime pDtmFechaCM) => Consultar("SELECT * FROM dbo.Proyecciones WHERE Fecha=@fecha ORDER BY Grupo,Codigo,Trabajadores", Parametro("@fecha", pDtmFechaCM.Date));
   /// <summary>Resume ventas reales del día y saldos pendientes; no mezcla insumos con unidades de pan.</summary>
   public DataTable Resumen(DateTime pDtmFechaCM) => Consultar("SELECT (SELECT COALESCE(SUM(PrecioTotal),0) FROM dbo.Ventas WHERE Fecha>=@fecha AND Fecha<DATEADD(day,1,@fecha)) AS VentasDelDia, (SELECT COALESCE(SUM(PrecioTotal-Pagado),0) FROM dbo.CuentasPorPagar) AS PendienteDePago", Parametro("@fecha", pDtmFechaCM.Date));
}
GRAFICAS
using System.Data;
using System.Drawing.Drawing2D;
 
namespace Panaderia.Presentacion;
 
/// <summary>Objetivo: representar un punto observado o calculado sin alterar datos de la base.</summary>
internal sealed record Punto(double pDblXCM, double pDblYCM);
/// <summary>Objetivo: identificar una curva y su color en la leyenda.</summary>
internal sealed record Curva(string pStrNombreCM, Color pObjColorCM, IReadOnlyList<Punto> pLstPuntosCM);
 
/// <summary>Objetivo: preparar gráficas comparables y estimar el equilibrio bajo costos unitarios constantes.</summary>
internal static class DatosGraficas
{
   /// <summary>Selecciona solo el mismo producto, fecha y serie; ordena por trabajadores sin mezclar unidades.</summary>
   public static DataRow[] Serie(DataRow pDtrSeleccionCM) => pDtrSeleccionCM.Table.AsEnumerable()
       .Where(pDtrFilaCM => pDtrFilaCM.Field<string>("Grupo") == pDtrSeleccionCM.Field<string>("Grupo")
           && pDtrFilaCM.Field<string>("Codigo") == pDtrSeleccionCM.Field<string>("Codigo")
           && Convert.ToDateTime(pDtrFilaCM["Fecha"]).Date == Convert.ToDateTime(pDtrSeleccionCM["Fecha"]).Date)
       .OrderBy(pDtrFilaCM => Convert.ToInt32(pDtrFilaCM["Trabajadores"])).ToArray();
 
   /// <summary>Convierte observaciones sin inventar un marginal para el primer punto nulo.</summary>
   public static Punto[] Puntos(DataRow[] pArrFilasCM, string pStrColumnaCM) => pArrFilasCM
       .Where(pDtrFilaCM => !pDtrFilaCM.IsNull(pStrColumnaCM))
       .Select(pDtrFilaCM => new Punto(Convert.ToDouble(pDtrFilaCM["Trabajadores"]), Convert.ToDouble(pDtrFilaCM[pStrColumnaCM]))).ToArray();
 
   /// <summary>Calcula intersección continua, costo variable unitario y dominio visible. Q=0 impide estimar CV unitario.</summary>
   public static (double? pDblEquilibrioCM, double pDblCostoUnitarioCM, double pDblMaximoCM) Equilibrio(DataRow pDtrFilaCM)
   {
       var lDblQCM = Convert.ToDouble(pDtrFilaCM["PFT"]);
       if (lDblQCM <= 0) return (null, 0, 0);
       var lDblCostoCM = Convert.ToDouble(pDtrFilaCM["CostoVariable"]) / lDblQCM;
       var lDblPrecioCM = Convert.ToDouble(pDtrFilaCM["Precio"]);
       var lDblFijoCM = Convert.ToDouble(pDtrFilaCM["CostoFijo"]);
       double? lDblEquilibrioCM = lDblPrecioCM > lDblCostoCM ? lDblFijoCM / (lDblPrecioCM - lDblCostoCM) : null;
       var lDblMaximoCM = Math.Max(1, Math.Max(lDblQCM, Convert.ToDouble(pDtrFilaCM["Demanda"])));
       if (lDblEquilibrioCM.HasValue) lDblMaximoCM = Math.Max(lDblMaximoCM, lDblEquilibrioCM.Value * 1.2);
       return (lDblEquilibrioCM, lDblCostoCM, lDblMaximoCM);
   }
}
 
/// <summary>Objetivo: mostrar producción, productividad y equilibrio sin escribir ni crear registros.</summary>
internal sealed class Graficas : Form
{
   /// <summary>Construye tres pestañas con curvas basadas en la selección actual de Proyecciones.</summary>
   public Graficas(DataRow pDtrSeleccionCM, string pStrUnidadCM)
   {
       Text = "Gráficas de Proyecciones"; ClientSize = new Size(1100, 700); MinimumSize = new Size(820, 560);
       StartPosition = FormStartPosition.CenterParent; Font = new Font("Segoe UI", 10); BackColor = Tematica.gColFondoCM; Icon = Tematica.Icono();
       Text = "El Horno de la 2 | Gráficas de Proyecciones";
       var lTabGraficasCM = new TabControl { Dock = DockStyle.Fill };
       Controls.Add(lTabGraficasCM);
       var lArrFilasCM = DatosGraficas.Serie(pDtrSeleccionCM);
       var lStrContextoCM = $"Serie: {pDtrSeleccionCM["Grupo"]}   |   Producto: {pDtrSeleccionCM["Codigo"]}   |   Fecha: {Convert.ToDateTime(pDtrSeleccionCM["Fecha"]):dd/MM/yyyy}   |   Unidad: {pStrUnidadCM}";
       var lStrNotaCM = lArrFilasCM.Length < 2 ? "Solo hay un registro: se muestra un punto. Registre más escenarios comparables para observar la evolución." : "Los puntos son registros observados o escenarios. Las líneas unen observaciones; no representan una curva continua estimada.";
       Agregar(lTabGraficasCM, "Producción total", lStrContextoCM,
           new Grafico("Trabajadores", $"PFT ({pStrUnidadCM})", [new("PFT", Color.RoyalBlue, DatosGraficas.Puntos(lArrFilasCM, "PFT"))]), lStrNotaCM);
       Agregar(lTabGraficasCM, "Producto medio y marginal", lStrContextoCM,
           new Grafico("Trabajadores", $"{pStrUnidadCM} por trabajador", [new("PFM", Color.RoyalBlue, DatosGraficas.Puntos(lArrFilasCM, "PFM")), new("PFMg", Color.DarkOrange, DatosGraficas.Puntos(lArrFilasCM, "PFMg"))]),
           lStrNotaCM + " El primer marginal nulo no se dibuja. Los valores negativos se conservan.");
       var lObjModeloCM = DatosGraficas.Equilibrio(pDtrSeleccionCM);
       var lDblPrecioCM = Convert.ToDouble(pDtrSeleccionCM["Precio"]);
       var lDblFijoCM = Convert.ToDouble(pDtrSeleccionCM["CostoFijo"]);
       var lStrEscenarioCM = lStrContextoCM + $"\nEscenario: {pDtrSeleccionCM["Trabajadores"]} trabajadores, PFT {pDtrSeleccionCM["PFT"]}, CF Q {lDblFijoCM:N2}, precio Q {lDblPrecioCM:N2}";
       var lLstCurvasCM = new List<Curva>();
       if (lObjModeloCM.pDblMaximoCM > 0)
       {
           var lDblMaxCM = lObjModeloCM.pDblMaximoCM;
           lLstCurvasCM.Add(new("Ingresos", Color.ForestGreen, [new(0, 0), new(lDblMaxCM, lDblMaxCM * lDblPrecioCM)]));
           lLstCurvasCM.Add(new("Costos totales", Color.Firebrick, [new(0, lDblFijoCM), new(lDblMaxCM, lDblFijoCM + lDblMaxCM * lObjModeloCM.pDblCostoUnitarioCM)]));
           if (lObjModeloCM.pDblEquilibrioCM is double lDblCruceCM)
               lLstCurvasCM.Add(new("Equilibrio", Color.DarkViolet, [new(lDblCruceCM, lDblCruceCM * lDblPrecioCM)]));
       }
       var lStrEquilibrioCM = lObjModeloCM.pDblMaximoCM == 0 ? "PFT es cero: no se puede calcular el costo variable unitario ni dibujar el equilibrio."
           : lObjModeloCM.pDblEquilibrioCM is double lDblPuntoCM
               ? $"Intersección: {lDblPuntoCM:N3} {pStrUnidadCM}. Umbral redondeado: {Math.Ceiling(lDblPuntoCM):N0}. CV unitario: Q {lObjModeloCM.pDblCostoUnitarioCM:N4}."
               : "Sin punto de equilibrio finito: el precio no supera el costo variable unitario.";
       Agregar(lTabGraficasCM, "Punto de equilibrio", lStrEscenarioCM,
           new Grafico($"Unidades vendidas ({pStrUnidadCM})", "Importe (Q)", lLstCurvasCM),
           lStrEquilibrioCM + "\nModelo lineal: ingreso = precio × unidades; costo = CF + CV unitario × unidades. Supone costos unitarios constantes y puede extenderse más allá de la capacidad y demanda registradas; no es una venta real.");
       foreach (TabPage lTabPaginaCM in lTabGraficasCM.TabPages) Tematica.Aplicar(lTabPaginaCM, Tematica.ColorModulo(4));
   }
   /// <summary>Distribuye contexto, gráfica y supuestos con espacio suficiente al redimensionar.</summary>
   private static void Agregar(TabControl pTabControlCM, string pStrTituloCM, string pStrContextoCM, Grafico pCtlGraficoCM, string pStrNotaCM)
   {
       var lTabPaginaCM = new TabPage(pStrTituloCM);
       var lPnlDisenoCM = new TableLayoutPanel { Dock = DockStyle.Fill, ColumnCount = 1, RowCount = 3, Padding = new Padding(10) };
       lPnlDisenoCM.RowStyles.Add(new RowStyle(SizeType.Absolute, 65));
       lPnlDisenoCM.RowStyles.Add(new RowStyle(SizeType.Percent, 100));
       lPnlDisenoCM.RowStyles.Add(new RowStyle(SizeType.Absolute, 105));
       lPnlDisenoCM.Controls.Add(new Label { Text = pStrContextoCM, Dock = DockStyle.Fill, AutoEllipsis = true }, 0, 0);
       lPnlDisenoCM.Controls.Add(pCtlGraficoCM, 0, 1);
       lPnlDisenoCM.Controls.Add(new Label { Text = pStrNotaCM, Dock = DockStyle.Fill }, 0, 2);
       lTabPaginaCM.Controls.Add(lPnlDisenoCM); pTabControlCM.TabPages.Add(lTabPaginaCM);
   }
}
 
/// <summary>Objetivo: dibujar curvas y puntos con ejes, escala, cuadrícula y leyenda sin dependencias adicionales.</summary>
internal sealed class Grafico : Control
{
   private readonly string gStrEjeXCM;
   private readonly string gStrEjeYCM;
   private readonly IReadOnlyList<Curva> gLstCurvasCM;
   /// <summary>Guarda series de solo lectura y activa doble búfer para redimensionar sin parpadeo.</summary>
   public Grafico(string pStrEjeXCM, string pStrEjeYCM, IReadOnlyList<Curva> pLstCurvasCM)
   {
       gStrEjeXCM = pStrEjeXCM; gStrEjeYCM = pStrEjeYCM; gLstCurvasCM = pLstCurvasCM;
       Dock = DockStyle.Fill; BackColor = Color.White; Font = new Font("Segoe UI", 10);
       SetStyle(ControlStyles.UserPaint | ControlStyles.AllPaintingInWmPaint | ControlStyles.OptimizedDoubleBuffer | ControlStyles.ResizeRedraw, true);
   }
   /// <summary>Dibuja únicamente datos finitos; incluye cero y valores negativos y evita escalas de amplitud cero.</summary>
   protected override void OnPaint(PaintEventArgs pObjEventoCM)
   {
       base.OnPaint(pObjEventoCM);
       var lObjDibujoCM = pObjEventoCM.Graphics; lObjDibujoCM.SmoothingMode = SmoothingMode.AntiAlias;
       var lArrPuntosCM = gLstCurvasCM.SelectMany(pObjCurvaCM => pObjCurvaCM.pLstPuntosCM).Where(pObjPuntoCM => double.IsFinite(pObjPuntoCM.pDblXCM) && double.IsFinite(pObjPuntoCM.pDblYCM)).ToArray();
       if (Width < 300 || Height < 180) return;
       lObjDibujoCM.DrawString(gStrEjeYCM, Font, Brushes.Black, 8, 8);
       if (lArrPuntosCM.Length == 0) { lObjDibujoCM.DrawString("No hay puntos disponibles para esta gráfica.", Font, Brushes.DimGray, 40, 70); return; }
       var lObjAreaCM = new RectangleF(100, 40, Width - 135, Height - 125);
       var lDblXMaxCM = Math.Max(1, lArrPuntosCM.Max(pObjPuntoCM => pObjPuntoCM.pDblXCM)) * 1.05;
       var lDblYMinCM = Math.Min(0, lArrPuntosCM.Min(pObjPuntoCM => pObjPuntoCM.pDblYCM));
       var lDblYMaxCM = Math.Max(0, lArrPuntosCM.Max(pObjPuntoCM => pObjPuntoCM.pDblYCM));
       if (lDblYMaxCM == lDblYMinCM) lDblYMaxCM = lDblYMinCM + 1;
       var lDblRangoCM = lDblYMaxCM - lDblYMinCM; lDblYMaxCM += lDblRangoCM * .08;
       if (lDblYMinCM < 0) lDblYMinCM -= lDblRangoCM * .08;
       PointF Posicion(Punto pObjPuntoCM) => new(lObjAreaCM.Left + (float)(pObjPuntoCM.pDblXCM / lDblXMaxCM * lObjAreaCM.Width), lObjAreaCM.Bottom - (float)((pObjPuntoCM.pDblYCM - lDblYMinCM) / (lDblYMaxCM - lDblYMinCM) * lObjAreaCM.Height));
       using var lObjCuadriculaCM = new Pen(Color.Gainsboro);
       using var lObjEjeCM = new Pen(Color.Gray);
       using var lObjFormatoCM = new StringFormat { Alignment = StringAlignment.Center };
       using var lObjDerechaCM = new StringFormat { Alignment = StringAlignment.Far };
       for (var lIntMarcaCM = 0; lIntMarcaCM <= 5; lIntMarcaCM++)
       {
           var lDblXCM = lDblXMaxCM * lIntMarcaCM / 5; var lDblYCM = lDblYMinCM + (lDblYMaxCM - lDblYMinCM) * lIntMarcaCM / 5;
           var lSngXCM = Posicion(new(lDblXCM, 0)).X; var lSngYCM = Posicion(new(0, lDblYCM)).Y;
           lObjDibujoCM.DrawLine(lObjCuadriculaCM, lSngXCM, lObjAreaCM.Top, lSngXCM, lObjAreaCM.Bottom);
           lObjDibujoCM.DrawLine(lObjCuadriculaCM, lObjAreaCM.Left, lSngYCM, lObjAreaCM.Right, lSngYCM);
           lObjDibujoCM.DrawString(Etiqueta(lDblXCM), Font, Brushes.Black, lSngXCM, lObjAreaCM.Bottom + 5, lObjFormatoCM);
           lObjDibujoCM.DrawString(Etiqueta(lDblYCM), Font, Brushes.Black, lObjAreaCM.Left - 8, lSngYCM - 8, lObjDerechaCM);
       }
       lObjDibujoCM.DrawLine(lObjEjeCM, lObjAreaCM.Left, lObjAreaCM.Top, lObjAreaCM.Left, lObjAreaCM.Bottom);
       lObjDibujoCM.DrawLine(lObjEjeCM, lObjAreaCM.Left, Posicion(new(0, 0)).Y, lObjAreaCM.Right, Posicion(new(0, 0)).Y);
       lObjDibujoCM.DrawString(gStrEjeXCM, Font, Brushes.Black, lObjAreaCM.Left + lObjAreaCM.Width / 2, lObjAreaCM.Bottom + 30, lObjFormatoCM);
       var lSngLeyendaCM = lObjAreaCM.Left;
       foreach (var lObjCurvaCM in gLstCurvasCM)
       {
           using var lObjTrazoCM = new Pen(lObjCurvaCM.pObjColorCM, 2.5f);
           using var lObjRellenoCM = new SolidBrush(lObjCurvaCM.pObjColorCM);
           var lArrSerieCM = lObjCurvaCM.pLstPuntosCM.Where(pObjPuntoCM => double.IsFinite(pObjPuntoCM.pDblXCM) && double.IsFinite(pObjPuntoCM.pDblYCM)).Select(Posicion).ToArray();
           if (lArrSerieCM.Length > 1) lObjDibujoCM.DrawLines(lObjTrazoCM, lArrSerieCM);
           foreach (var lObjPuntoCM in lArrSerieCM) lObjDibujoCM.FillEllipse(lObjRellenoCM, lObjPuntoCM.X - 4, lObjPuntoCM.Y - 4, 8, 8);
           lObjDibujoCM.DrawLine(lObjTrazoCM, lSngLeyendaCM, Height - 20, lSngLeyendaCM + 22, Height - 20);
           lObjDibujoCM.DrawString(lObjCurvaCM.pStrNombreCM, Font, Brushes.Black, lSngLeyendaCM + 28, Height - 29);
           lSngLeyendaCM += 45 + lObjDibujoCM.MeasureString(lObjCurvaCM.pStrNombreCM, Font).Width;
       }
   }
   /// <summary>Evita desbordamiento de etiquetas con cifras grandes y conserva precisión para escalas pequeñas.</summary>
   private static string Etiqueta(double pDblValorCM) => Math.Abs(pDblValorCM) >= 1000000 ? pDblValorCM.ToString("0.##E+0") : pDblValorCM.ToString("0.##");
}
 
CAPA PRESENTACION
using System.Data;
using System.Globalization;
using Panaderia.Entidades;
using Panaderia.LogicaNegocio;
 
namespace Panaderia.Presentacion;
 
/// <summary>Objetivo: presentar inventario, ventas, obligaciones, clientes y análisis de producción en cinco pestañas.</summary>
internal sealed class Principal : Form
{
   private readonly Servicio gObjServicioCM;
   private readonly TabControl gTabModulosCM = new Navegacion();
   private readonly DataGridView gDgvStockCM = Tabla();
   private readonly DataGridView gDgvDisponibleCM = Tabla();
   private readonly DataGridView gDgvCarritoCM = Tabla();
   private readonly DataGridView gDgvCuentasCM = Tabla();
   private readonly DataGridView gDgvClientesCM = Tabla();
   private readonly DataGridView gDgvHistorialCM = Tabla();
   private readonly DataGridView gDgvProyeccionesCM = Tabla();
   private readonly List<LineaVenta> gLstCarritoCM = [];
   private readonly DataTable gDtbCarritoCM = new();
   private readonly TextBox gTxtNitCM = Texto(20);
   private readonly TextBox gTxtNombreCM = Texto(80);
   private readonly TextBox gTxtApellidoCM = Texto(80);
   private readonly TextBox gTxtCiudadCM = Texto(80);
   private readonly TextBox gTxtDireccionCM = Texto(200);
   private readonly Label gLblTotalCM = new() { AutoSize = true };
   private readonly Label gLblResumenCM = new() { AutoSize = true };
   private readonly DateTimePicker gDtpHistorialCM = Fecha();
   private readonly DateTimePicker gDtpProyeccionCM = Fecha();
   private readonly TextBox gTxtFiltroNitCM = Texto(20);
 
   /// <summary>Inicializa módulos; verifica conexión al mostrar el formulario.</summary>
   public Principal(Servicio pObjServicioCM, bool pBlnConsultarAlMostrarCM = true)
   {
       gObjServicioCM = pObjServicioCM;
       Text = "El Horno de la 2 | Gestión de panadería"; ClientSize = new Size(1280, 900);
       MinimumSize = new Size(1150, 800); StartPosition = FormStartPosition.CenterScreen; WindowState = FormWindowState.Maximized;
       Font = new Font("Segoe UI", 10); BackColor = Tematica.gColFondoCM; Icon = Tematica.Icono();
       var lPnlMarcoCM = new TableLayoutPanel { Dock = DockStyle.Fill, ColumnCount = 1, RowCount = 3, Margin = Padding.Empty };
       lPnlMarcoCM.RowStyles.Add(new RowStyle(SizeType.Absolute, 96));
       lPnlMarcoCM.RowStyles.Add(new RowStyle(SizeType.Percent, 100));
       lPnlMarcoCM.RowStyles.Add(new RowStyle(SizeType.Absolute, 50));
       lPnlMarcoCM.Controls.Add(Tematica.Cabecera(), 0, 0); lPnlMarcoCM.Controls.Add(gTabModulosCM, 0, 1);
       lPnlMarcoCM.Controls.Add(Tematica.Credito(), 0, 2); Controls.Add(lPnlMarcoCM);
       gDtbCarritoCM.Columns.Add("Codigo"); gDtbCarritoCM.Columns.Add("Producto"); gDtbCarritoCM.Columns.Add("Unidad");
       gDtbCarritoCM.Columns.Add("Cantidad", typeof(decimal)); gDtbCarritoCM.Columns.Add("Precio", typeof(decimal)); gDtbCarritoCM.Columns.Add("Total", typeof(decimal));
       gDgvCarritoCM.DataSource = gDtbCarritoCM;
       Inventario(); Ventas(); Cuentas(); Clientes(); Proyecciones();
       for (var lIntModuloCM = 0; lIntModuloCM < gTabModulosCM.TabCount; lIntModuloCM++)
           Tematica.Aplicar(gTabModulosCM.TabPages[lIntModuloCM], Tematica.ColorModulo(lIntModuloCM));
       if (pBlnConsultarAlMostrarCM)
       {
           Shown += (pObjRemitenteCM, pObjEventoCM) => Seguro(ActualizarTodo);
           gTabModulosCM.SelectedIndexChanged += (pObjRemitenteCM, pObjEventoCM) => Seguro(ActualizarTodo);
       }
   }
   /// <summary>Crea tablas solo de lectura; las escrituras pasan por la lógica de negocio.</summary>
   private static DataGridView Tabla() => new() { Dock = DockStyle.Fill, ReadOnly = true, AllowUserToAddRows = false, AllowUserToDeleteRows = false, MultiSelect = false, SelectionMode = DataGridViewSelectionMode.FullRowSelect, AutoSizeColumnsMode = DataGridViewAutoSizeColumnsMode.DisplayedCells, BackgroundColor = Color.White, RowHeadersVisible = false };
   /// <summary>Crea un campo textual con el mismo límite que su columna SQL.</summary>
   private static TextBox Texto(int pIntLongitudCM) => new() { Width = 155, MaxLength = pIntLongitudCM };
   /// <summary>Crea un campo decimal no negativo; entradas/salidas usan una casilla adicional.</summary>
   private static NumericUpDown Numero(int pIntDecimalesCM = 3) => new() { Width = 130, DecimalPlaces = pIntDecimalesCM, Maximum = 100000000, ThousandsSeparator = true };
   /// <summary>Crea un selector de fecha sin hora.</summary>
   private static DateTimePicker Fecha() => new() { Width = 160, Format = DateTimePickerFormat.Short };
   /// <summary>Crea una lista cerrada para categorías válidas.</summary>
   private static ComboBox Lista(params string[] pArrOpcionesCM)
   { var lCmbListaCM = new ComboBox { Width = 150, DropDownStyle = ComboBoxStyle.DropDownList }; lCmbListaCM.Items.AddRange(pArrOpcionesCM); lCmbListaCM.SelectedIndex = 0; return lCmbListaCM; }
   /// <summary>Crea una barra de captura con desplazamiento si la ventana es pequeña.</summary>
   private static FlowLayoutPanel Barra() => new() { Dock = DockStyle.Fill, AutoScroll = true, Padding = new Padding(8), WrapContents = true };
   /// <summary>Agrupa etiqueta y campo para conservar legibilidad.</summary>
   private static void Campo(FlowLayoutPanel pPnlPanelCM, string pStrEtiquetaCM, Control pCtlCampoCM)
   {
       var lPnlGrupoCM = new FlowLayoutPanel { FlowDirection = FlowDirection.TopDown, AutoSize = true, WrapContents = false, Margin = new Padding(5) };
       lPnlGrupoCM.Controls.Add(new Label { Text = pStrEtiquetaCM, AutoSize = true }); lPnlGrupoCM.Controls.Add(pCtlCampoCM); pPnlPanelCM.Controls.Add(lPnlGrupoCM);
   }
   /// <summary>Agrega una acción y captura errores de validación o conexión.</summary>
   private void Boton(FlowLayoutPanel pPnlPanelCM, string pStrTextoCM, Action pActAccionCM)
   { var lBtnAccionCM = new Button { Text = pStrTextoCM, AutoSize = true, Height = 36, Margin = new Padding(5, 25, 5, 5) }; lBtnAccionCM.Click += (pObjRemitenteCM, pObjEventoCM) => Seguro(pActAccionCM); pPnlPanelCM.Controls.Add(lBtnAccionCM); }
   /// <summary>Muestra mensajes de error sin cerrar la aplicación ni anunciar éxitos inexistentes.</summary>
   private void Seguro(Action pActAccionCM)
   { try { pActAccionCM(); } catch (Exception lObjErrorCM) { MessageBox.Show(this, lObjErrorCM.Message + "\nSi es un error de conexión, revise conexion.txt y ejecute Panaderia.sql.", "Operación no completada", MessageBoxButtons.OK, MessageBoxIcon.Warning); } }
   /// <summary>Crea una pestaña con contenido expansible y botones de limpieza y cierre siempre visibles.</summary>
   private TableLayoutPanel Modulo(string pStrTituloCM, params int[] pArrAlturasCM)
   {
       var lTabPaginaCM = new TabPage(pStrTituloCM) { Padding = new Padding(10), UseVisualStyleBackColor = false, Font = new Font("Segoe UI", 10) };
       var lPnlDisenoCM = new TableLayoutPanel { Dock = DockStyle.Fill, ColumnCount = 1, RowCount = pArrAlturasCM.Length + 1, BackColor = Tematica.gColFondoCM };
       foreach (var lIntAlturaCM in pArrAlturasCM) lPnlDisenoCM.RowStyles.Add(new RowStyle(SizeType.Absolute, lIntAlturaCM));
       lPnlDisenoCM.RowStyles.Add(new RowStyle(SizeType.Percent, 100));
       var lPnlContenedorCM = new TableLayoutPanel { Dock = DockStyle.Fill, ColumnCount = 1, RowCount = 2 };
       lPnlContenedorCM.RowStyles.Add(new RowStyle(SizeType.Percent, 100));
       lPnlContenedorCM.RowStyles.Add(new RowStyle(SizeType.Absolute, 70));
       var lPnlBotonesCM = Barra(); lPnlBotonesCM.WrapContents = false;
       Boton(lPnlBotonesCM, "Limpiar", () =>
       {
           LimpiarCampos(lPnlDisenoCM);
           if (pStrTituloCM == "Ventas") VaciarCarrito();
       });
       Boton(lPnlBotonesCM, "Finalizar", Close);
       foreach (Control lCtlBotonCM in lPnlBotonesCM.Controls) lCtlBotonCM.Margin = new Padding(5);
       lPnlContenedorCM.Controls.Add(lPnlDisenoCM, 0, 0);
       lPnlContenedorCM.Controls.Add(lPnlBotonesCM, 0, 1);
       lTabPaginaCM.Controls.Add(lPnlContenedorCM);
       gTabModulosCM.TabPages.Add(lTabPaginaCM); return lPnlDisenoCM;
   }
   /// <summary>Restablece capturas de una pestaña: textos vacíos, números al mínimo, listas al primer valor y fechas a hoy. Conserva los datos guardados y las tablas.</summary>
   private static void LimpiarCampos(Control pCtlContenedorCM)
   {
       foreach (Control lCtlCampoCM in pCtlContenedorCM.Controls)
       {
           switch (lCtlCampoCM)
           {
               case TextBox lTxtCampoCM:
                   lTxtCampoCM.Clear(); break;
               case NumericUpDown lNumCampoCM:
                   lNumCampoCM.Value = lNumCampoCM.Minimum; break;
               case ComboBox lCmbCampoCM:
                   lCmbCampoCM.SelectedIndex = lCmbCampoCM.Items.Count > 0 ? 0 : -1; break;
               case DateTimePicker lDtpCampoCM:
                   lDtpCampoCM.Value = DateTime.Today;
                   if (lDtpCampoCM.ShowCheckBox) lDtpCampoCM.Checked = false;
                   break;
               case CheckBox lChkCampoCM:
                   lChkCampoCM.Checked = false; break;
               case DataGridView:
                   break;
               default:
                   if (lCtlCampoCM.HasChildren) LimpiarCampos(lCtlCampoCM);
                   break;
           }
       }
   }
   /// <summary>Obtiene una selección válida de la tabla.</summary>
   private static DataRow Fila(DataGridView pDgvTablaCM) => (pDgvTablaCM.CurrentRow?.DataBoundItem as DataRowView)?.Row ?? throw new ArgumentException("Seleccione una fila de la tabla.");
 
   /// <summary>Objetivo: altas de productos, entradas y consumo/merma, además de precios y ubicación.</summary>
   private void Inventario()
   {
       var lPnlDisenoCM = Modulo("Inventario", 235, 115); var lPnlAltaCM = Barra(); var lPnlMovimientoCM = Barra();
       var lTxtCodigoCM = Texto(30); var lTxtProductoCM = Texto(100); var lCmbCategoriaCM = Lista("Terminado", "Insumo"); var lCmbUnidadCM = Lista("Unidad", "Fila", "Libra", "Kilogramo", "Litro");
       var lNumCantidadCM = Numero(); var lNumPrecioCM = Numero(2); var lTxtUbicacionCM = Texto(100); var lDtpCreacionCM = Fecha(); var lDtpVenceCM = Fecha(); lDtpVenceCM.ShowCheckBox = true; lDtpVenceCM.Checked = false;
       Campo(lPnlAltaCM,"Código",lTxtCodigoCM); Campo(lPnlAltaCM,"Producto",lTxtProductoCM); Campo(lPnlAltaCM,"Categoría",lCmbCategoriaCM); Campo(lPnlAltaCM,"Unidad de stock y venta",lCmbUnidadCM); Campo(lPnlAltaCM,"Saldo inicial",lNumCantidadCM); Campo(lPnlAltaCM,"Precio por unidad (Q)",lNumPrecioCM); Campo(lPnlAltaCM,"Ubicación",lTxtUbicacionCM); Campo(lPnlAltaCM,"Creación",lDtpCreacionCM); Campo(lPnlAltaCM,"Vence (opcional)",lDtpVenceCM);
       Boton(lPnlAltaCM,"Crear producto",() => { gObjServicioCM.CrearProducto(new(lTxtCodigoCM.Text.Trim(),lTxtProductoCM.Text.Trim(),lCmbCategoriaCM.Text,lCmbUnidadCM.Text,lNumCantidadCM.Value,lNumPrecioCM.Value,lTxtUbicacionCM.Text,lDtpCreacionCM.Value,lDtpVenceCM.Checked?lDtpVenceCM.Value:null)); ActualizarTodo(); });
       Boton(lPnlAltaCM,"Precio/ubicación de selección",() => { gObjServicioCM.ActualizarPrecio(Fila(gDgvStockCM)["Codigo"].ToString()!,lNumPrecioCM.Value,lTxtUbicacionCM.Text); ActualizarTodo(); });
       var lNumMovimientoCM = Numero(); var lCmbSentidoCM = Lista("Entrada", "Salida"); var lTxtMotivoCM = Texto(200);
       Campo(lPnlMovimientoCM,"Cantidad seleccionada",lNumMovimientoCM); Campo(lPnlMovimientoCM,"Movimiento",lCmbSentidoCM); Campo(lPnlMovimientoCM,"Motivo / consumo / merma",lTxtMotivoCM);
       Boton(lPnlMovimientoCM,"Registrar movimiento",() => { gObjServicioCM.Movimiento(Fila(gDgvStockCM)["Codigo"].ToString()!,lNumMovimientoCM.Value*(lCmbSentidoCM.Text=="Salida"?-1:1),lTxtMotivoCM.Text); ActualizarTodo(); });
       Boton(lPnlMovimientoCM,"Actualizar",ActualizarTodo);
       lPnlDisenoCM.Controls.Add(lPnlAltaCM,0,0); lPnlDisenoCM.Controls.Add(lPnlMovimientoCM,0,1); lPnlDisenoCM.Controls.Add(gDgvStockCM,0,2);
   }
   /// <summary>Objetivo: buscar/crear cliente, armar carrito y confirmar venta atómica.</summary>
   private void Ventas()
   {
       var lPnlDisenoCM = Modulo("Ventas",140,135,95); var lPnlClienteCM = Barra(); var lPnlAccionesCM = Barra();
       Campo(lPnlClienteCM,"1. NIT",gTxtNitCM); Boton(lPnlClienteCM,"Buscar NIT",BuscarNit);
       Campo(lPnlClienteCM,"Nombre",gTxtNombreCM); Campo(lPnlClienteCM,"Apellido",gTxtApellidoCM); Campo(lPnlClienteCM,"Ciudad",gTxtCiudadCM); Campo(lPnlClienteCM,"Dirección",gTxtDireccionCM);
       Boton(lPnlClienteCM,"Guardar cliente",() => { gObjServicioCM.GuardarCliente(new(gTxtNitCM.Text,gTxtNombreCM.Text,gTxtApellidoCM.Text,gTxtCiudadCM.Text,gTxtDireccionCM.Text)); MessageBox.Show("Cliente guardado."); ActualizarTodo(); });
       var lNumCantidadCM = Numero(); var lCmbPagoCM = Lista("Efectivo","Tarjeta","Transferencia"); Campo(lPnlAccionesCM,"Cantidad en unidad mostrada",lNumCantidadCM); Campo(lPnlAccionesCM,"Forma de pago",lCmbPagoCM);
       Boton(lPnlAccionesCM,"Agregar selección",() => AgregarCarrito(lNumCantidadCM.Value));
       Boton(lPnlAccionesCM,"Vaciar carrito",VaciarCarrito);
       Boton(lPnlAccionesCM,"Confirmar venta",() => { var lIntVentaCM = gObjServicioCM.Vender(gTxtNitCM.Text,lCmbPagoCM.Text,gLstCarritoCM); VaciarCarrito(); ActualizarTodo(); MessageBox.Show($"Venta {lIntVentaCM} registrada y stock descontado."); });
       lPnlAccionesCM.Controls.Add(gLblTotalCM);
       lPnlDisenoCM.Controls.Add(lPnlClienteCM,0,0); lPnlDisenoCM.Controls.Add(gDgvDisponibleCM,0,1); lPnlDisenoCM.Controls.Add(lPnlAccionesCM,0,2); lPnlDisenoCM.Controls.Add(gDgvCarritoCM,0,3);
   }
   /// <summary>Busca NIT y limpia los campos si no existe para evitar reutilizar datos de otro cliente.</summary>
   private void BuscarNit()
   {
       var lDtbClienteCM = gObjServicioCM.BuscarCliente(gTxtNitCM.Text);
       gTxtNombreCM.Clear(); gTxtApellidoCM.Clear(); gTxtCiudadCM.Clear(); gTxtDireccionCM.Clear();
       if(lDtbClienteCM.Rows.Count==0) { MessageBox.Show("NIT nuevo: complete nombre y dirección, luego guarde el cliente."); return; }
       var lDtrClienteCM=lDtbClienteCM.Rows[0]; gTxtNombreCM.Text=lDtrClienteCM["Nombre"].ToString(); gTxtApellidoCM.Text=lDtrClienteCM["Apellido"].ToString(); gTxtCiudadCM.Text=lDtrClienteCM["Ciudad"].ToString(); gTxtDireccionCM.Text=lDtrClienteCM["Direccion"].ToString();
   }
   /// <summary>Acumula cantidades por código y presenta el total estimado con precio visible.</summary>
   private void AgregarCarrito(decimal pDecCantidadCM)
   {
       if(pDecCantidadCM<=0) throw new ArgumentException("Ingrese una cantidad positiva.");
       var lDtrProductoCM=Fila(gDgvDisponibleCM); var lStrCodigoCM=lDtrProductoCM["Codigo"].ToString()!;
       var lIntIndiceCM=gLstCarritoCM.FindIndex(pObjLineaCM=>pObjLineaCM.pStrCodigoCM==lStrCodigoCM);
       var lDecCantidadCM=pDecCantidadCM+(lIntIndiceCM>=0?gLstCarritoCM[lIntIndiceCM].pDecCantidadCM:0);
       if(lDecCantidadCM>Convert.ToDecimal(lDtrProductoCM["Cantidad"])) throw new ArgumentException("Cantidad acumulada mayor al stock visible.");
       var lDecPrecioCM=Convert.ToDecimal(lDtrProductoCM["PrecioUnidad"]);
       if(lIntIndiceCM>=0) { gLstCarritoCM[lIntIndiceCM]=new(lStrCodigoCM,lDecCantidadCM); gDtbCarritoCM.Rows[lIntIndiceCM]["Cantidad"]=lDecCantidadCM; gDtbCarritoCM.Rows[lIntIndiceCM]["Total"]=Math.Round(lDecCantidadCM*lDecPrecioCM,2,MidpointRounding.AwayFromZero); }
       else { gLstCarritoCM.Add(new(lStrCodigoCM,lDecCantidadCM)); gDtbCarritoCM.Rows.Add(lStrCodigoCM,lDtrProductoCM["Producto"],lDtrProductoCM["Unidad"],lDecCantidadCM,lDecPrecioCM,Math.Round(lDecCantidadCM*lDecPrecioCM,2,MidpointRounding.AwayFromZero)); }
       gLblTotalCM.Text=$"Total estimado: Q {gDtbCarritoCM.AsEnumerable().Sum(pDtrFilaCM=>pDtrFilaCM.Field<decimal>("Total")):N2}";
   }
   /// <summary>Limpia las líneas únicamente después de éxito o por solicitud del usuario.</summary>
   private void VaciarCarrito() { gLstCarritoCM.Clear(); gDtbCarritoCM.Rows.Clear(); gLblTotalCM.Text="Total: Q 0.00"; }
   /// <summary>Objetivo: registrar obligaciones y abonos parciales.</summary>
   private void Cuentas()
   {
       var lPnlDisenoCM=Modulo("Cuentas x pagar",235,110); var lPnlAltaCM=Barra(); var lPnlPagoCM=Barra();
       var lCmbTipoCM=Lista("Proveedor","Nomina","Gasto fijo"); var lTxtProveedorCM=Texto(100); var lTxtProductoCM=Texto(100); var lTxtPedidoCM=Texto(50); var lNumCantidadCM=Numero(); var lNumPrecioCM=Numero(2); var lDtpPedidoCM=Fecha(); var lDtpEntregaCM=Fecha(); var lTxtRecibeCM=Texto(100);
       Campo(lPnlAltaCM,"Tipo",lCmbTipoCM); Campo(lPnlAltaCM,"Proveedor / empleado",lTxtProveedorCM); Campo(lPnlAltaCM,"Producto / concepto",lTxtProductoCM); Campo(lPnlAltaCM,"Pedido / referencia",lTxtPedidoCM); Campo(lPnlAltaCM,"Cantidad",lNumCantidadCM); Campo(lPnlAltaCM,"Precio unitario (Q)",lNumPrecioCM); Campo(lPnlAltaCM,"Fecha pedido",lDtpPedidoCM); Campo(lPnlAltaCM,"Entrega / vencimiento pago",lDtpEntregaCM); Campo(lPnlAltaCM,"Quién recibe",lTxtRecibeCM);
       Boton(lPnlAltaCM,"Guardar obligación",()=> {gObjServicioCM.GuardarCuenta(new(lCmbTipoCM.Text,lTxtProveedorCM.Text,lTxtProductoCM.Text,lTxtPedidoCM.Text,lNumCantidadCM.Value,lNumPrecioCM.Value,lDtpPedidoCM.Value,lDtpEntregaCM.Value,lTxtRecibeCM.Text)); ActualizarTodo();});
       var lNumAbonoCM=Numero(2); Campo(lPnlPagoCM,"Abono a cuenta seleccionada",lNumAbonoCM); Boton(lPnlPagoCM,"Registrar abono",()=>{gObjServicioCM.Abonar(Convert.ToInt32(Fila(gDgvCuentasCM)["IdCuenta"]),lNumAbonoCM.Value); ActualizarTodo();});
       lPnlDisenoCM.Controls.Add(lPnlAltaCM,0,0); lPnlDisenoCM.Controls.Add(lPnlPagoCM,0,1); lPnlDisenoCM.Controls.Add(gDgvCuentasCM,0,2);
   }
   /// <summary>Objetivo: consultar clientes y compras en un día concreto, con filtro opcional por NIT.</summary>
   private void Clientes()
   {
       var lPnlDisenoCM=Modulo("Clientes",90,200); var lPnlFiltroCM=Barra(); Campo(lPnlFiltroCM,"Fecha de compras",gDtpHistorialCM); Campo(lPnlFiltroCM,"NIT (vacío = todos)",gTxtFiltroNitCM); Boton(lPnlFiltroCM,"Consultar",ActualizarTodo);
       lPnlDisenoCM.Controls.Add(lPnlFiltroCM,0,0); lPnlDisenoCM.Controls.Add(gDgvClientesCM,0,1); lPnlDisenoCM.Controls.Add(gDgvHistorialCM,0,2);
   }
   /// <summary>Objetivo: registrar ensayos comparables, producción real y sus costos para analizar microeconomía.</summary>
   private void Proyecciones()
   {
       var lPnlDisenoCM=Modulo("Proyecciones",275,100); var lPnlCapturaCM=Barra(); var lPnlNotaCM=Barra();
       var lTxtGrupoCM=Texto(50); var lTxtCodigoCM=Texto(30); var lNumTrabajadoresCM=Numero(0); lNumTrabajadoresCM.Minimum=1; lNumTrabajadoresCM.Value=1;
       var lNumQCM=Numero(); var lNumCfCM=Numero(2); var lNumCvCM=Numero(2); var lNumPrecioCM=Numero(2); var lNumDemandaCM=Numero(); var lChkRealCM=new CheckBox{Text="Producción real: aumenta stock",AutoSize=true};
       Campo(lPnlCapturaCM,"Fecha",gDtpProyeccionCM); Campo(lPnlCapturaCM,"Serie (igual capital/horas)",lTxtGrupoCM); Campo(lPnlCapturaCM,"Código producto terminado",lTxtCodigoCM); Campo(lPnlCapturaCM,"Trabajadores",lNumTrabajadoresCM); Campo(lPnlCapturaCM,"PFT (unidad del producto)",lNumQCM); Campo(lPnlCapturaCM,"Costo fijo diario (Q)",lNumCfCM); Campo(lPnlCapturaCM,"Costo variable TOTAL (Q)",lNumCvCM); Campo(lPnlCapturaCM,"Precio venta por unidad (Q)",lNumPrecioCM); Campo(lPnlCapturaCM,"Demanda prevista (unidades)",lNumDemandaCM); Campo(lPnlCapturaCM,"Tipo de registro",lChkRealCM);
       Boton(lPnlCapturaCM,"Registrar",()=> { gObjServicioCM.Produccion(new(lTxtGrupoCM.Text,lTxtCodigoCM.Text,(int)lNumTrabajadoresCM.Value,lNumQCM.Value,lNumCfCM.Value,lNumCvCM.Value,lNumPrecioCM.Value,lNumDemandaCM.Value,lChkRealCM.Checked,gDtpProyeccionCM.Value)); ActualizarTodo(); }); Boton(lPnlCapturaCM,"Consultar fecha",ActualizarTodo);
       Boton(lPnlCapturaCM, "Ver gráficas", MostrarGraficas);
       lPnlNotaCM.Controls.Add(new Label {AutoSize=true,MaximumSize=new Size(1100,0),Text="PFM = Q/L; PFMg = ΔQ/ΔL entre puntos comparables. Etapas aproximadas. Equilibrio = CF/(precio−CV/Q).\nDemanda es una estimación ingresada por usted. Consumo de ingredientes: registrar salidas en Inventario. No duplicar producción real como entrada manual."}); lPnlNotaCM.Controls.Add(gLblResumenCM);
       lPnlDisenoCM.Controls.Add(lPnlCapturaCM,0,0); lPnlDisenoCM.Controls.Add(lPnlNotaCM,0,1); lPnlDisenoCM.Controls.Add(gDgvProyeccionesCM,0,2);
   }
   /// <summary>Abre gráficas de la serie del registro seleccionado usando únicamente datos ya consultados.</summary>
   private void MostrarGraficas()
   {
       var lDtrSeleccionCM = Fila(gDgvProyeccionesCM);
       var lStrCodigoCM = lDtrSeleccionCM["Codigo"].ToString()!;
       var lStrUnidadCM = "unidad del producto";
       if (gDgvStockCM.DataSource is DataTable lDtbStockCM)
       {
           var lDtrProductoCM = lDtbStockCM.AsEnumerable().FirstOrDefault(pDtrFilaCM => pDtrFilaCM["Codigo"].ToString() == lStrCodigoCM);
           if (lDtrProductoCM != null) lStrUnidadCM = lDtrProductoCM["Unidad"].ToString()!;
       }
       using var lFrmGraficasCM = new Graficas(lDtrSeleccionCM, lStrUnidadCM);
       lFrmGraficasCM.ShowDialog(this);
   }
   /// <summary>Actualiza vistas desde SQL para reflejar ventas, producción y pagos confirmados.</summary>
   private void ActualizarTodo()
   {
       gObjServicioCM.ProbarConexion(); gDgvStockCM.DataSource=gObjServicioCM.Inventario(); gDgvDisponibleCM.DataSource=gObjServicioCM.Inventario(true);
       gDgvCuentasCM.DataSource=gObjServicioCM.Cuentas(); gDgvClientesCM.DataSource=gObjServicioCM.Clientes(); gDgvHistorialCM.DataSource=gObjServicioCM.Historial(gDtpHistorialCM.Value,gTxtFiltroNitCM.Text);
       gDgvProyeccionesCM.DataSource=gObjServicioCM.Proyecciones(gDtpProyeccionCM.Value);
       var lDtrResumenCM=gObjServicioCM.Resumen(gDtpProyeccionCM.Value).Rows[0]; gLblResumenCM.Text=$"Ventas reales de la fecha: Q {Convert.ToDecimal(lDtrResumenCM["VentasDelDia"]):N2} | Cuentas pendientes: Q {Convert.ToDecimal(lDtrResumenCM["PendienteDePago"]):N2}";
   }
}
PROGRAMA
using Panaderia.LogicaNegocio;
 
namespace Panaderia.Presentacion;
/// <summary>Objetivo: iniciar Windows Forms y exigir login antes de abrir los módulos.</summary>
internal static class Program
{
   /// <summary>Carga configuración externa e inicia el ciclo de mensajes de Windows.</summary>
   [STAThread]
   private static void Main()
   {
       ApplicationConfiguration.Initialize();
       try
       {
           var lStrConexionCM = File.ReadAllText(Path.Combine(AppContext.BaseDirectory, "conexion.txt")).Trim();
           var lObjServicioCM = new Servicio(lStrConexionCM);
           using var lFrmLoginCM = new Login(lObjServicioCM);
           if (lFrmLoginCM.ShowDialog() == DialogResult.OK) Application.Run(new Principal(lObjServicioCM));
       }
       catch (Exception lObjErrorCM) { MessageBox.Show(lObjErrorCM.Message, "No fue posible iniciar"); }
   }
}
 
/// <summary>Objetivo: solicitar las credenciales del proyecto académico.</summary>
internal sealed class Login : Form
{
   /// <summary>Construye el formulario y oculta los caracteres de contraseña.</summary>
   public Login(Servicio pObjServicioCM)
   {
       Text = "El Horno de la 2 | Iniciar sesión"; ClientSize = new Size(920, 600); StartPosition = FormStartPosition.CenterScreen;
       FormBorderStyle = FormBorderStyle.FixedDialog; MaximizeBox = false;
       Font = new Font("Segoe UI", 10); BackColor = Tematica.gColFondoCM; Icon = Tematica.Icono();
       var lPnlMarcoCM = new TableLayoutPanel { Dock = DockStyle.Fill, ColumnCount = 1, RowCount = 2, Margin = Padding.Empty };
       lPnlMarcoCM.RowStyles.Add(new RowStyle(SizeType.Percent, 100)); lPnlMarcoCM.RowStyles.Add(new RowStyle(SizeType.Absolute, 64));
       var lPnlContenidoCM = new TableLayoutPanel { Dock = DockStyle.Fill, ColumnCount = 2, RowCount = 1, Margin = Padding.Empty };
       lPnlContenidoCM.ColumnStyles.Add(new ColumnStyle(SizeType.Percent, 42)); lPnlContenidoCM.ColumnStyles.Add(new ColumnStyle(SizeType.Percent, 58));
       var lPnlMarcaCM = new TableLayoutPanel { Dock = DockStyle.Fill, ColumnCount = 1, RowCount = 4, BackColor = Color.FromArgb(240, 229, 208), Padding = new Padding(22), Margin = Padding.Empty };
       lPnlMarcaCM.RowStyles.Add(new RowStyle(SizeType.Percent, 60)); lPnlMarcaCM.RowStyles.Add(new RowStyle(SizeType.Absolute, 65)); lPnlMarcaCM.RowStyles.Add(new RowStyle(SizeType.Absolute, 38)); lPnlMarcaCM.RowStyles.Add(new RowStyle(SizeType.Percent, 40));
       lPnlMarcaCM.Controls.Add(new PictureBox { Image = Tematica.Imagen("logo"), Dock = DockStyle.Fill, SizeMode = PictureBoxSizeMode.Zoom, Margin = new Padding(12, 22, 12, 2) }, 0, 0);
       lPnlMarcaCM.Controls.Add(new Label { Text = "EL HORNO DE LA 2", Dock = DockStyle.Fill, TextAlign = ContentAlignment.MiddleCenter, ForeColor = Tematica.gColTextoCM, Font = new Font("Georgia", 17, FontStyle.Bold) }, 0, 1);
       lPnlMarcaCM.Controls.Add(new Label { Text = "P A N A D E R Í A", Dock = DockStyle.Fill, TextAlign = ContentAlignment.MiddleCenter, ForeColor = Tematica.gColCafeCM, Font = new Font("Segoe UI", 12) }, 0, 2);
       lPnlMarcaCM.Controls.Add(new Label { Text = "Tradición en cada detalle\n\nGestión y análisis de producción", Dock = DockStyle.Fill, TextAlign = ContentAlignment.MiddleCenter, ForeColor = Tematica.gColCafeCM }, 0, 3);
       var lPnlPanelCM = new FlowLayoutPanel { Dock = DockStyle.Fill, FlowDirection = FlowDirection.TopDown, Padding = new Padding(48, 55, 24, 24), WrapContents = false, BackColor = Color.FromArgb(255, 252, 246), Margin = Padding.Empty };
       var lTxtUsuarioCM = new TextBox { Width = 390, PlaceholderText = "Ingrese su usuario", Font = new Font("Segoe UI", 12), BorderStyle = BorderStyle.FixedSingle, Margin = new Padding(0, 4, 0, 16) };
       var lTxtClaveCM = new TextBox { Width = 390, PlaceholderText = "Ingrese su contraseña", UseSystemPasswordChar = true, Font = new Font("Segoe UI", 12), BorderStyle = BorderStyle.FixedSingle, Margin = new Padding(0, 4, 0, 24) };
       var lBtnEntrarCM = new Button { Text = "Ingresar", Width = 390, Height = 46, Margin = new Padding(0, 4, 0, 12) };
       Tematica.Boton(lBtnEntrarCM, Tematica.gColCafeCM);
       lBtnEntrarCM.Click += (pObjRemitenteCM, pObjEventoCM) =>
       {
           if (pObjServicioCM.Login(lTxtUsuarioCM.Text, lTxtClaveCM.Text)) { DialogResult = DialogResult.OK; Close(); }
           else MessageBox.Show("Usuario o contraseña incorrectos.");
       };
       lPnlPanelCM.Controls.Add(new Label { Text = "Bienvenido", AutoSize = true, ForeColor = Tematica.gColTextoCM, Font = new Font("Segoe UI", 27, FontStyle.Bold), Margin = new Padding(0, 0, 0, 8) });
       lPnlPanelCM.Controls.Add(new Label { Text = "Ingrese para administrar su panadería.", AutoSize = true, ForeColor = Tematica.gColCafeCM, Margin = new Padding(0, 0, 0, 28) });
       lPnlPanelCM.Controls.Add(new Label { Text = "USUARIO", AutoSize = true, ForeColor = Tematica.gColCafeCM });
       lPnlPanelCM.Controls.Add(lTxtUsuarioCM);
       lPnlPanelCM.Controls.Add(new Label { Text = "CONTRASEÑA", AutoSize = true, ForeColor = Tematica.gColCafeCM });
       lPnlPanelCM.Controls.Add(lTxtClaveCM); lPnlPanelCM.Controls.Add(lBtnEntrarCM);
       lPnlPanelCM.Controls.Add(new Label { Text = "Inventario · Ventas · Clientes\nCuentas por pagar · Proyecciones", AutoSize = true, ForeColor = Tematica.gColCafeCM, Font = new Font("Segoe UI", 9), Margin = new Padding(0, 10, 0, 0) });
       lPnlContenidoCM.Controls.Add(lPnlMarcaCM, 0, 0); lPnlContenidoCM.Controls.Add(lPnlPanelCM, 1, 0);
       lPnlMarcoCM.Controls.Add(lPnlContenidoCM, 0, 0); lPnlMarcoCM.Controls.Add(Tematica.Credito(), 0, 1);
       Controls.Add(lPnlMarcoCM); AcceptButton = lBtnEntrarCM;
   }
}
 
TEMATICA
using System.Reflection;
 
namespace Panaderia.Presentacion;
 
/// <summary>Objetivo: centralizar la identidad El Horno de la 2, los colores, los iconos y los créditos académicos.</summary>
internal static class Tematica
{
   public const string gStrMarcaCM = "El Horno de la 2";
   public const string gStrCreditoCM = "Modelado, creado y diseñado por estudiantes de la sección A de Ingeniería de Sistemas de la Universidad Mariano Gálvez.";
   public static readonly Color gColFondoCM = Color.FromArgb(247, 243, 235);
   public static readonly Color gColTextoCM = Color.FromArgb(65, 48, 37);
   public static readonly Color gColCafeCM = Color.FromArgb(102, 67, 43);
   public static readonly Color gColDoradoCM = Color.FromArgb(181, 118, 41);
   private static readonly Dictionary<string, Image> gDicImagenesCM = new();
 
   /// <summary>Carga recursos incrustados para que el ejecutable no dependa de rutas externas de imágenes.</summary>
   public static Image Imagen(string pStrNombreCM)
   {
       if (gDicImagenesCM.TryGetValue(pStrNombreCM, out var lObjExistenteCM)) return lObjExistenteCM;
       var lObjEnsambladoCM = Assembly.GetExecutingAssembly();
       var lStrRecursoCM = lObjEnsambladoCM.GetManifestResourceNames().Single(pStrRutaCM => pStrRutaCM.EndsWith(".Assets." + pStrNombreCM + ".png", StringComparison.Ordinal));
       using var lObjFlujoCM = lObjEnsambladoCM.GetManifestResourceStream(lStrRecursoCM)!;
       using var lObjOriginalCM = Image.FromStream(lObjFlujoCM);
       var lObjImagenCM = new Bitmap(lObjOriginalCM);
       gDicImagenesCM.Add(pStrNombreCM, lObjImagenCM); return lObjImagenCM;
   }
   /// <summary>Obtiene el icono de aplicación desde los recursos del ensamblado.</summary>
   public static Icon Icono()
   {
       var lObjEnsambladoCM = Assembly.GetExecutingAssembly();
       using var lObjFlujoCM = lObjEnsambladoCM.GetManifestResourceStream(lObjEnsambladoCM.GetManifestResourceNames().Single(pStrRutaCM => pStrRutaCM.EndsWith(".Assets.espiga.ico")))!;
       using var lObjIconoCM = new Icon(lObjFlujoCM); return (Icon)lObjIconoCM.Clone();
   }
   /// <summary>Asigna un color estable a cada módulo para facilitar su reconocimiento.</summary>
   public static Color ColorModulo(int pIntIndiceCM) => pIntIndiceCM switch
   {
       0 => gColDoradoCM, 1 => Color.FromArgb(40, 115, 107), 2 => Color.FromArgb(156, 76, 76),
       3 => Color.FromArgb(79, 112, 152), _ => Color.FromArgb(117, 97, 142)
   };
   /// <summary>Asocia cada módulo con un pictograma original.</summary>
   public static string ImagenModulo(int pIntIndiceCM) => pIntIndiceCM switch { 0 => "inventory", 1 => "sales", 2 => "payments", 3 => "clients", _ => "projections" };
 
   /// <summary>Crea la cabecera de marca con emblema y descripción funcional.</summary>
   public static Control Cabecera()
   {
       var lPnlCabeceraCM = new TableLayoutPanel { Dock = DockStyle.Fill, ColumnCount = 3, RowCount = 1, BackColor = Color.FromArgb(255, 252, 246), Padding = new Padding(18, 8, 18, 8), Margin = Padding.Empty };
       lPnlCabeceraCM.ColumnStyles.Add(new ColumnStyle(SizeType.Absolute, 90));
       lPnlCabeceraCM.ColumnStyles.Add(new ColumnStyle(SizeType.Percent, 100));
       lPnlCabeceraCM.ColumnStyles.Add(new ColumnStyle(SizeType.Absolute, 215));
       lPnlCabeceraCM.Controls.Add(new PictureBox { Image = Imagen("logo"), Dock = DockStyle.Fill, SizeMode = PictureBoxSizeMode.Zoom, Margin = new Padding(0, 0, 12, 0) }, 0, 0);
       var lPnlTitulosCM = new TableLayoutPanel { Dock = DockStyle.Fill, RowCount = 2, ColumnCount = 1, Margin = Padding.Empty };
       lPnlTitulosCM.RowStyles.Add(new RowStyle(SizeType.Percent, 60)); lPnlTitulosCM.RowStyles.Add(new RowStyle(SizeType.Percent, 40));
       lPnlTitulosCM.Controls.Add(new Label { Text = "EL HORNO DE LA 2", Dock = DockStyle.Fill, TextAlign = ContentAlignment.BottomLeft, ForeColor = gColTextoCM, Font = new Font("Georgia", 23, FontStyle.Bold) }, 0, 0);
       lPnlTitulosCM.Controls.Add(new Label { Text = "PANADERÍA  /  Gestión y análisis de producción", Dock = DockStyle.Fill, TextAlign = ContentAlignment.MiddleLeft, ForeColor = gColCafeCM, Font = new Font("Segoe UI", 10) }, 0, 1);
       lPnlCabeceraCM.Controls.Add(lPnlTitulosCM, 1, 0);
       lPnlCabeceraCM.Controls.Add(new Label { Text = "PROYECTO ACADÉMICO\nMicroeconomía · Ingeniería de Sistemas", Dock = DockStyle.Fill, ForeColor = gColCafeCM, TextAlign = ContentAlignment.MiddleRight, Font = new Font("Segoe UI", 9) }, 2, 0);
       return lPnlCabeceraCM;
   }
   /// <summary>Presenta el crédito estudiantil como texto, sin utilizar escudos o marcas institucionales.</summary>
   public static Control Credito() => new Label { Text = gStrCreditoCM, Dock = DockStyle.Fill, TextAlign = ContentAlignment.MiddleCenter, Padding = new Padding(18, 6, 18, 6), BackColor = Color.FromArgb(235, 226, 211), ForeColor = gColCafeCM, Font = new Font("Segoe UI", 9), Margin = Padding.Empty };
 
   /// <summary>Aplica estilo y pictograma a una acción, conservando el texto y su evento original.</summary>
   public static void Boton(Button pBtnBotonCM, Color pColAcentoCM)
   {
       var lStrTextoCM = pBtnBotonCM.Text;
       var lStrIconoCM = lStrTextoCM.Contains("gráficas") ? "charts" : lStrTextoCM.Contains("Buscar") || lStrTextoCM.Contains("Consultar") ? "search"
           : lStrTextoCM.Contains("Limpiar") || lStrTextoCM.Contains("Vaciar") ? "clear" : lStrTextoCM.Contains("Finalizar") ? "exit"
           : lStrTextoCM.Contains("Actualizar") ? "refresh" : lStrTextoCM.Contains("Precio") ? "edit" : lStrTextoCM.Contains("Ingresar") ? "lock"
           : lStrTextoCM.Contains("Crear") || lStrTextoCM.Contains("Agregar") ? "add" : "save";
       pBtnBotonCM.FlatStyle = FlatStyle.Flat; pBtnBotonCM.FlatAppearance.BorderSize = 0;
       pBtnBotonCM.BackColor = lStrTextoCM == "Finalizar" ? Color.FromArgb(151, 69, 57) : pColAcentoCM;
       pBtnBotonCM.ForeColor = Color.White; pBtnBotonCM.UseVisualStyleBackColor = false;
       pBtnBotonCM.Font = new Font("Segoe UI", 9.5f, FontStyle.Bold);
       pBtnBotonCM.Padding = new Padding(9, 4, 9, 4); pBtnBotonCM.MinimumSize = new Size(0, 38);
       pBtnBotonCM.Cursor = Cursors.Hand;
       pBtnBotonCM.Image = new Bitmap(Imagen(lStrIconoCM), new Size(19, 19));
       pBtnBotonCM.TextImageRelation = TextImageRelation.ImageBeforeText;
       pBtnBotonCM.ImageAlign = ContentAlignment.MiddleLeft;
       pBtnBotonCM.FlatAppearance.MouseOverBackColor = ControlPaint.Light(pBtnBotonCM.BackColor, .12f);
       pBtnBotonCM.FlatAppearance.MouseDownBackColor = ControlPaint.Dark(pBtnBotonCM.BackColor, .12f);
   }
 
   /// <summary>Configura capturas y tablas con contraste, filas alternas y encabezados del color del módulo.</summary>
   public static void Aplicar(Control pCtlRaizCM, Color pColAcentoCM)
   {
       pCtlRaizCM.ForeColor = gColTextoCM;
       switch (pCtlRaizCM)
       {
           case Button lBtnCampoCM: Boton(lBtnCampoCM, pColAcentoCM); return;
           case DataGridView lDgvTablaCM:
               lDgvTablaCM.EnableHeadersVisualStyles = false; lDgvTablaCM.BorderStyle = BorderStyle.None;
               lDgvTablaCM.BackgroundColor = Color.White; lDgvTablaCM.GridColor = Color.FromArgb(232, 225, 214);
               lDgvTablaCM.ColumnHeadersBorderStyle = DataGridViewHeaderBorderStyle.Single;
               lDgvTablaCM.ColumnHeadersDefaultCellStyle = new DataGridViewCellStyle { BackColor = pColAcentoCM, ForeColor = Color.White, Font = new Font("Segoe UI", 9.5f, FontStyle.Bold), Padding = new Padding(7, 4, 7, 4), SelectionBackColor = pColAcentoCM, SelectionForeColor = Color.White };
               lDgvTablaCM.ColumnHeadersHeightSizeMode = DataGridViewColumnHeadersHeightSizeMode.AutoSize;
               lDgvTablaCM.DefaultCellStyle = new DataGridViewCellStyle { BackColor = Color.White, ForeColor = gColTextoCM, Font = new Font("Segoe UI", 9.5f), SelectionBackColor = Color.FromArgb(232, 223, 204), SelectionForeColor = gColTextoCM, Padding = new Padding(6, 3, 6, 3) };
               lDgvTablaCM.AlternatingRowsDefaultCellStyle.BackColor = Color.FromArgb(251, 248, 242);
               lDgvTablaCM.RowTemplate.Height = 32; return;
           case TextBox lTxtCampoCM: lTxtCampoCM.BorderStyle = BorderStyle.FixedSingle; lTxtCampoCM.BackColor = Color.FromArgb(255, 254, 251); break;
           case ComboBox lCmbCampoCM: lCmbCampoCM.FlatStyle = FlatStyle.Flat; lCmbCampoCM.BackColor = Color.FromArgb(255, 254, 251); break;
           case NumericUpDown lNumCampoCM: lNumCampoCM.BorderStyle = BorderStyle.FixedSingle; lNumCampoCM.BackColor = Color.FromArgb(255, 254, 251); return;
           case FlowLayoutPanel: pCtlRaizCM.BackColor = Color.White; break;
           case TabPage: pCtlRaizCM.BackColor = gColFondoCM; break;
       }
       foreach (Control lCtlHijoCM in pCtlRaizCM.Controls) Aplicar(lCtlHijoCM, pColAcentoCM);
   }
}
 
/// <summary>Objetivo: dibujar las cinco opciones con icono, fondo y acento individual, manteniendo navegación por teclado.</summary>
internal sealed class Navegacion : TabControl
{
   /// <summary>Activa pestañas anchas con dibujo propio y texto legible.</summary>
   public Navegacion()
   {
       Dock = DockStyle.Fill; DrawMode = TabDrawMode.OwnerDrawFixed; SizeMode = TabSizeMode.Fixed;
       ItemSize = new Size(215, 54); Padding = new Point(10, 5); Font = new Font("Segoe UI", 10, FontStyle.Bold);
   }
   /// <summary>Dibuja el estado activo y una franja de identificación para cada módulo.</summary>
   protected override void OnDrawItem(DrawItemEventArgs pObjEventoCM)
   {
       var lIntIndiceCM = pObjEventoCM.Index; if (lIntIndiceCM < 0) return;
       var lObjAreaCM = pObjEventoCM.Bounds; var lColAcentoCM = Tematica.ColorModulo(lIntIndiceCM);
       using var lObjFondoCM = new SolidBrush(SelectedIndex == lIntIndiceCM ? Color.White : Color.FromArgb(238, 231, 219));
       using var lObjAcentoCM = new SolidBrush(lColAcentoCM);
       pObjEventoCM.Graphics.FillRectangle(lObjFondoCM, lObjAreaCM);
       pObjEventoCM.Graphics.FillRectangle(lObjAcentoCM, lObjAreaCM.Left, lObjAreaCM.Bottom - 4, lObjAreaCM.Width, 4);
       pObjEventoCM.Graphics.DrawImage(Tematica.Imagen(Tematica.ImagenModulo(lIntIndiceCM)), lObjAreaCM.Left + 14, lObjAreaCM.Top + 13, 27, 27);
       TextRenderer.DrawText(pObjEventoCM.Graphics, TabPages[lIntIndiceCM].Text, Font, new Rectangle(lObjAreaCM.Left + 49, lObjAreaCM.Top, lObjAreaCM.Width - 53, lObjAreaCM.Height - 4), Tematica.gColTextoCM, TextFormatFlags.Left | TextFormatFlags.VerticalCenter | TextFormatFlags.EndEllipsis);
       if ((pObjEventoCM.State & DrawItemState.Focus) != 0) pObjEventoCM.DrawFocusRectangle();
   }
}
 