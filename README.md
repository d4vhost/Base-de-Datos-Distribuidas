CREATE DATABASE flotilla;
GO
USE flotilla;
GO

CREATE TABLE Conductor (
    nombre VARCHAR(80) NOT NULL,
    licenciaVigente BIT NOT NULL,
    telefono CHAR(15) NOT NULL,
    curp VARCHAR(20) NOT NULL,
    disponibilidad BIT NOT NULL DEFAULT 1,
    PRIMARY KEY (curp)
);

CREATE TABLE Combustible (
    idCombustible INT NOT NULL IDENTITY(1,1),
    tipo VARCHAR(80) NOT NULL,
    precioPorLitro DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (idCombustible)
);

CREATE TABLE Vehiculo (
    modelo VARCHAR(80) NOT NULL,
    placa VARCHAR(15) NOT NULL,
    anio INT NOT NULL,
    disponibilidad BIT NOT NULL DEFAULT 1,
    marca VARCHAR(80) NOT NULL,
    capacidadCombustible DECIMAL(10,2) NOT NULL,
    seguro VARCHAR(80) NOT NULL,
    idCombustible INT NOT NULL,
    PRIMARY KEY (placa),
    FOREIGN KEY (idCombustible) REFERENCES Combustible (idCombustible)
);

CREATE TABLE Mantenimiento (
    idMantenimiento INT NOT NULL IDENTITY(1,1),
    fecha DATE NOT NULL,
    costo DECIMAL(10,2) NOT NULL,
    descripcion VARCHAR(100) NOT NULL,
    placa VARCHAR(15) NOT NULL,
    PRIMARY KEY (idMantenimiento),
    FOREIGN KEY (placa) REFERENCES Vehiculo(placa)
);

CREATE TABLE Ruta (
    idRuta INT NOT NULL IDENTITY(1,1),
    kilometraje DECIMAL(10,2) NOT NULL,
    destino VARCHAR(80) NOT NULL,
    origen VARCHAR(80) NOT NULL,
    fecha DATE NOT NULL,
    horaSalida TIME NOT NULL,
    horaLlegada TIME NOT NULL,
    placa VARCHAR(15) NOT NULL,
    curp VARCHAR(20) NOT NULL,
    PRIMARY KEY (idRuta),
    FOREIGN KEY (placa) REFERENCES Vehiculo(placa),
    FOREIGN KEY (curp) REFERENCES Conductor (curp)
);



-- Insertar datos en Combustible
INSERT INTO Combustible (tipo, precioPorLitro) VALUES 
('Magna', 24.79),
('Premium', 25.55),
('Diésel', 26.80);

-- Insertar datos en Conductor
INSERT INTO Conductor (nombre, licenciaVigente, telefono, curp, disponibilidad) VALUES 
('Alejandro Gonzalez Cruz', 1, '5546986133', 'GOCA040227HDFMZSA1', 1),
('Ruben Ruiz Diaz', 1, '5546236512', 'RUDR040227HDFMZSA1', 1),
('Luis Torres Lopez', 0, '5598683214', 'TOLL040227HDFMZSA1', 1),
('Antonio Cruz Rosas', 1, '5590436712', 'CRRA040227HDFMZSA1', 1),
('Pedro Fuentes Herrera', 0, '5553681243', 'FUHP040227HDFMZSA1', 1);

-- Insertar datos en Vehiculo
INSERT INTO Vehiculo (modelo, placa, anio, disponibilidad, marca, capacidadCombustible, seguro, idCombustible) VALUES 
('Rifter', 'AS12-AS3', 2020, 1, 'Peugeot', 20.00, 'Qualitas', 1),
('Saveiro', 'FDS32-12', 2021, 1, 'Volkswagen', 18.00, 'Qualitas', 2),
('Oroch', 'FD3-45G', 2020, 1, 'Renault', 16.00, 'GNP', 3),
('RAM 1200', 'FDL-42K', 2023, 1, 'RAM', 14.00, 'GNP', 1),
('Rifter', 'SDF-12', 2024, 1, 'Peugeot', 15.00, 'Qualitas', 2);

-- Insertar datos en Ruta
INSERT INTO Ruta (kilometraje, destino, origen, fecha, horaSalida, horaLlegada, placa, curp) VALUES 
(200, 'Cuautitlán', 'Toreo', '2025-02-07', '17:55:59', '21:55:59', 'AS12-AS3', 'GOCA040227HDFMZSA1'),
(100, 'Coapa', 'Toreo', '2025-02-06', '14:31:45', '15:55:59', 'FDS32-12', 'RUDR040227HDFMZSA1'),
(40, 'Santa Fe', 'Toreo', '2025-02-06', '13:55:12', '14:55:59', 'FD3-45G', 'TOLL040227HDFMZSA1'),
(50, 'Lomas Verdes', 'San Mateo', '2025-02-08', '18:51:39', '19:55:59', 'FDL-42K', 'CRRA040227HDFMZSA1'),
(60, 'Satélite', 'San Mateo', '2025-02-08', '19:52:13', '20:55:59', 'SDF-12', 'FUHP040227HDFMZSA1');

-- Insertar datos en Mantenimiento
INSERT INTO Mantenimiento (fecha, costo, descripcion, placa) VALUES 
('2024-10-12', 5000, 'Servicio completo', 'AS12-AS3'),
('2024-12-20', 3000, 'Servicio parcial', 'FDS32-12'),
('2025-01-10', 4000, 'Servicio completo', 'FD3-45G'),
('2024-08-29', 3500, 'Servicio parcial', 'FDL-42K'),
('2025-02-01', 4500, 'Servicio completo', 'SDF-12');


#Trigger del control de vencimiento de licencia de un conductor
CREATE TRIGGER trg_CheckLicenseExpiration
ON CONDUCTORES
AFTER INSERT, UPDATE
AS
BEGIN
    DECLARE @FEC_VEN_LIC DATE;
    DECLARE @NOM_CON VARCHAR(10);
    SELECT @FEC_VEN_LIC = i.FEC_VEN_LIC, @NOM_CON = i.NOM_CON
    FROM inserted i;
    IF @FEC_VEN_LIC <= GETDATE()
    BEGIN
        RAISERROR ('La licencia del conductor %s está vencida y no se puede insertar o actualizar el registro.', 16, 1, @NOM_CON);
        ROLLBACK TRANSACTION;
    END
END;

#trigger para el control y disponibilidad de vehiculos
CREATE TRIGGER trg_CheckVehicleAssignment
ON CONDUCTORES
AFTER INSERT, UPDATE
AS
BEGIN
    DECLARE @ID_VEH_CON INT;
    DECLARE @ESTADO_DISP CHAR(1);
    DECLARE @NOM_CON VARCHAR(10);
    SELECT @ID_VEH_CON = i.ID_VEH_CON, @NOM_CON = i.NOM_CON
    FROM inserted i;
    SELECT @ESTADO_DISP = v.ESTADO_DISP
    FROM VEHICULO v
    WHERE v.ID_VEH = @ID_VEH_CON;
    IF @ESTADO_DISP = 'N'
    BEGIN
        RAISERROR ('El vehículo asignado al conductor %s no está disponible y no se puede asignar.', 16, 1, @NOM_CON);
        ROLLBACK TRANSACTION;
    END
END;
INSERT INTO CONDUCTORES (NOM_CON, LIC_CON, FEC_VEN_LIC, ID_VEH_CON) VALUES
('Luis', 'PQR678', '2025-12-31', 2);


#trigger para ver el costo del viaje
CREATE TRIGGER trg_CalcularCostoViaje
ON Ruta
AFTER INSERT, UPDATE
AS
BEGIN
    DECLARE @kilometraje DECIMAL(10,2);
    DECLARE @idCombustible INT;
    DECLARE @precioPorLitro DECIMAL(10,2);
    DECLARE @costo DECIMAL(10,2);
    DECLARE @placa VARCHAR(15);
    SELECT @kilometraje = i.kilometraje, @placa = i.placa
    FROM inserted i;
    SELECT @idCombustible = v.idCombustible
    FROM Vehiculo v
    WHERE v.placa = @placa;
    SELECT @precioPorLitro = c.precioPorLitro
    FROM Combustible c
    WHERE c.idCombustible = @idCombustible;
    SET @costo = @kilometraje * @precioPorLitro;
    PRINT 'El costo estimado del viaje es: ' + CAST(@costo AS VARCHAR(20));
END;

INSERT INTO Conductor (nombre, licenciaVigente, telefono, curp, disponibilidad) VALUES 
('Juan Perez', 0, '5551234567', 'PEJU040227HDFMZSA1', 1);


Practica Base de datos

REVISAR NOMBRE DE USUARIO DE WINDOWS
C:\Users\d4vho>echo %USERNAME%
d4vho

Abre la ventana de Servicios:
  Presiona Win + R para abrir la ventana de Ejecutar.
  Escribe services.msc y presiona Enter.

Busca el servicio OpenSSH SSH Server:
  En la lista de servicios, busca OpenSSH SSH Server.
  Verifica que el estado del servicio sea Iniciado. Si no está iniciado, haz clic           derecho sobre el servicio y selecciona Iniciar.

Permitir SSH a través del firewall:
 Abre el Panel de control y ve a Sistema y seguridad > Firewall de Windows Defender >   Permitir una aplicación o característica a través del Firewall de Windows Defender.
Asegúrate de que OpenSSH Server esté marcado para las redes privadas y públicas.

DESACTIVAR FIREWALL DE WINDOWS
COMANDO DE COPIAR Y PEGAR ARCHIVO A WINDOWS
scp /home/d4vhost/Escritorio/data.odt d4vho@192.168.10.2:/Users/d4vho/Desktop

IP UBUNTU
192.168.10.3

IP WINDOWS 
192.168.10.2

###########CREACION DE INSTANCIAS EN SQL SERVER - UBUNTU ########
sqlcmd -S localhost -U sa -P 'sqlMyadmin' -C
1>CREATE DATABASE INSTANCIA_A
2>GO
3>SELECT NAME FROM sys.databases;

1>SELECT * FROM INSTANCIA_A.dbo.Estudiantes_Todos;
2>go

CERACION DE VISTAS
CREATE TABLE Estudiantes_Quito ( 

    id INT PRIMARY KEY, 

    nombre VARCHAR(50), 

    carrera VARCHAR(50), 

    ciudad VARCHAR(50) 

); 

  

-- Insertamos las filas correspondientes 

INSERT INTO Estudiantes_Quito VALUES 

(1, 'Ana Pérez', 'Ingeniería', 'Quito'), 

(3, 'Carla Ruiz', 'Ingeniería', 'Quito'); 

 Fragmento 2 (Estudiantes_Ambato): 

 

-- Fragmento Ambato 

CREATE TABLE Estudiantes_Ambato ( 

    id INT PRIMARY KEY, 

    nombre VARCHAR(50), 

    carrera VARCHAR(50), 

    ciudad VARCHAR(50) 

); 

  

INSERT INTO Estudiantes_Ambato VALUES 

(2, 'Luis Mora', 'Medicina', 'Ambato'), 

(5, 'Rosa Vega', 'Medicina', 'Ambato'); 


 Fragmento 3 (Estudiantes_Cuenca): 

 

-- Fragmento Cuenca 

CREATE TABLE Estudiantes_Cuenca ( 

    id INT PRIMARY KEY, 

    nombre VARCHAR(50), 

    carrera VARCHAR(50), 

    ciudad VARCHAR(50) 

); 

  

INSERT INTO Estudiantes_Cuenca VALUES 

(4, 'Mario León', 'Derecho', 'Cuenca'), 

(6, 'J. Ortega', 'Derecho', 'Cuenca'); 

 Si quisiéramos consultar toda la tabla original, podríamos usar una vista como esta: 

 

CREATE VIEW Estudiantes_Todos AS 

SELECT * FROM Estudiantes_Quito 

UNION ALL 

SELECT * FROM Estudiantes_Ambato 

UNION ALL 

SELECT * FROM Estudiantes_Cuenca; 






