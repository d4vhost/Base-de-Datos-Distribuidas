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




