# DQL - Data Query Language

# Introdução

A Linguagem de Consulta de Dados (DQL) é um subconjunto do SQL usado para realizar consultas em bancos de dados. No contexto do nosso jogo, as consultas DQL são essenciais para recuperar informações sobre o estado do jogo, como a localização do jogador, seu inventário, missões disponíveis e status dos NPCs.

## Scripts SQL

Funções e triggers foram reunidas em um único arquivo:
[`sql/dql.sql`](../../sql/dql.sql). Os trechos abaixo ilustram cada
parte presente nesse script.

### V24__CriarPersonagem_funcao.sql
```sql
CREATE OR REPLACE FUNCTION criar_personagem(
    p_nome_personagem VARCHAR(100)
) RETURNS TEXT AS $$
DECLARE
    v_sala_inicial RECORD;
    v_id_estagiario INT;
BEGIN

    SELECT id_sala, id_andar INTO v_sala_inicial
    FROM vw_sala_central --uma view
    WHERE numero_andar = 0;

    INSERT INTO Estagiario(
        nome, 
        xp, 
        nivel, 
        respeito,
        coins,
        status,
        andar_atual,
        sala_atual
    ) VALUES (
        p_nome_personagem,
        0,      
        1,    
        50,    
        100,   
        'Normal',
        v_sala_inicial.id_andar,
        v_sala_inicial.id_sala
    ) RETURNING id_personagem INTO v_id_estagiario;

    RETURN 'Estagiário "' || p_nome_personagem || '" criado com sucesso! (ID: ' || v_id_estagiario || ')';

EXCEPTION
    WHEN unique_violation THEN
        RETURN 'Erro: Já existe um estagiário com este nome.';
    WHEN OTHERS THEN
        RETURN 'Erro inesperado ao criar estagiário: ' || SQLERRM;
END;
$$ LANGUAGE plpgsql;


CREATE OR REPLACE FUNCTION criar_inventario_estagiario()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO Inventario (id_estagiario, espaco_total)
    VALUES (NEW.id_personagem, 12);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_criar_inventario
AFTER INSERT ON Estagiario
FOR EACH ROW
EXECUTE FUNCTION criar_inventario_estagiario();

```

### V25__LocalDetalhado_funcao.sql
```sql
CREATE OR REPLACE FUNCTION descrever_local_detalhado(
    p_id_personagem INT
) RETURNS TABLE(nome_local TEXT, descricao TEXT, saidas TEXT[]) AS $$
DECLARE
    v_posicao RECORD;
BEGIN
    
    SELECT * INTO v_posicao
    FROM vw_posicao_estagiario --uma view
    WHERE id_personagem = p_id_personagem;

    RETURN QUERY
    WITH saidas_possiveis AS (
        
        SELECT 'Ir para ' || nome_sala_destino as direcao
        FROM vw_conexoes_salas
        WHERE id_sala_origem = v_posicao.sala_atual

        UNION ALL

     
        SELECT 
            CASE 
                WHEN a.numero > v_posicao.numero_andar THEN 'Subir para ' || REGEXP_REPLACE(a.nome, '.*?: ', '')
                WHEN a.numero < v_posicao.numero_andar THEN 'Descer para ' || REGEXP_REPLACE(a.nome, '.*?: ', '')
            END
        FROM Andar a
        WHERE v_posicao.nome_sala = 'Sala Central'
        AND ((a.numero = v_posicao.numero_andar + 1 AND v_posicao.numero_andar < 10)
          OR (a.numero = v_posicao.numero_andar - 1 AND v_posicao.numero_andar > -2))
    )
    SELECT 
        (v_posicao.nome_andar || E'\nSala: ' || v_posicao.nome_sala),
        v_posicao.descricao_sala,
        ARRAY(
            SELECT direcao 
            FROM saidas_possiveis 
            WHERE direcao IS NOT NULL
            ORDER BY direcao
        ); -- esse text é para transformar em um vetor as saidas
END;
$$ LANGUAGE plpgsql;
```

### V26__MoverPersonagem_funcao.sql
```sql
CREATE OR REPLACE FUNCTION mover_personagem(
    p_id_personagem INT,
    p_direcao_texto VARCHAR(100)
) RETURNS TEXT AS $$
DECLARE
    v_posicao RECORD;
    v_novo_andar INT;
    v_nova_sala INT;
BEGIN

    SELECT * INTO v_posicao
    FROM vw_posicao_estagiario --uma view
    WHERE id_personagem = p_id_personagem;

    IF (p_direcao_texto LIKE 'Subir para%' OR p_direcao_texto LIKE 'Descer para%') THEN

        IF v_posicao.nome_sala != 'Sala Central' THEN
            RETURN 'Você só pode mudar de andar a partir da Sala Central.';
        END IF;


        SELECT sc.id_andar, sc.id_sala 
        INTO v_novo_andar, v_nova_sala
        FROM vw_sala_central sc
        JOIN Andar a ON sc.id_andar = a.id_andar
        WHERE p_direcao_texto LIKE '%' || REGEXP_REPLACE(a.nome, '.*?: ', '');

        IF v_novo_andar IS NOT NULL THEN
            UPDATE Estagiario 
            SET andar_atual = v_novo_andar,
                sala_atual = v_nova_sala
            WHERE id_personagem = p_id_personagem;
            
            RETURN 'Você mudou de andar.';
        END IF;

    ELSIF p_direcao_texto LIKE 'Ir para%' THEN

        SELECT id_sala_destino INTO v_nova_sala
        FROM vw_conexoes_salas
        WHERE id_sala_origem = v_posicao.sala_atual
        AND 'Ir para ' || nome_sala_destino = p_direcao_texto;

        IF v_nova_sala IS NOT NULL THEN
            UPDATE Estagiario 
            SET sala_atual = v_nova_sala
            WHERE id_personagem = p_id_personagem;
            
            RETURN 'Você se moveu para outra sala.';
        END IF;
    END IF;

    RETURN 'Movimento inválido.';
END;
$$ LANGUAGE plpgsql;
```

### V31__VerificarItem_trigger.sql
```sql
CREATE OR REPLACE FUNCTION verificar_item_tipo_unico() RETURNS TRIGGER AS $$
DECLARE
    tipo_item TEXT;
    count_tipos INT := 0;
BEGIN
    SELECT tipo INTO tipo_item FROM Item WHERE id_item = NEW.id_item;

    IF tipo_item = 'PowerUp' THEN
        SELECT COUNT(*) INTO count_tipos FROM Consumivel WHERE id_item = NEW.id_item;
        IF count_tipos > 0 THEN
            RAISE EXCEPTION 'Item do tipo PowerUp não pode estar também como Consumivel.';
        END IF;

        SELECT COUNT(*) INTO count_tipos FROM Equipamento WHERE id_item = NEW.id_item;
        IF count_tipos > 0 THEN
            RAISE EXCEPTION 'Item do tipo PowerUp não pode estar também como Equipamento.';
        END IF;
    ELSIF tipo_item = 'Consumivel' THEN
        SELECT COUNT(*) INTO count_tipos FROM PowerUp WHERE id_item = NEW.id_item;
        IF count_tipos > 0 THEN
            RAISE EXCEPTION 'Item do tipo Consumivel não pode estar também como PowerUp.';
        END IF;

        SELECT COUNT(*) INTO count_tipos FROM Equipamento WHERE id_item = NEW.id_item;
        IF count_tipos > 0 THEN
            RAISE EXCEPTION 'Item do tipo Consumivel não pode estar também como Equipamento.';
        END IF;
    ELSIF tipo_item = 'Equipamento' THEN
        SELECT COUNT(*) INTO count_tipos FROM PowerUp WHERE id_item = NEW.id_item;
        IF count_tipos > 0 THEN
            RAISE EXCEPTION 'Item do tipo Equipamento não pode estar também como PowerUp.';
        END IF;

        SELECT COUNT(*) INTO count_tipos FROM Consumivel WHERE id_item = NEW.id_item;
        IF count_tipos > 0 THEN
            RAISE EXCEPTION 'Item do tipo Equipamento não pode estar também como Consumivel.';
        END IF;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- PowerUp
CREATE TRIGGER validar_tipo_powerup
BEFORE INSERT OR UPDATE ON PowerUp
FOR EACH ROW
EXECUTE FUNCTION verificar_item_tipo_unico();

-- Consumivel
CREATE TRIGGER validar_tipo_consumivel
BEFORE INSERT OR UPDATE ON Consumivel
FOR EACH ROW
EXECUTE FUNCTION verificar_item_tipo_unico();

-- Equipamento
CREATE TRIGGER validar_tipo_equipamento
BEFORE INSERT OR UPDATE ON Equipamento
FOR EACH ROW
EXECUTE FUNCTION verificar_item_tipo_unico();

```

## Outras buscas importantes
```sql
-- 1.localização do jogador
SELECT 
    estagiario.id_personagem,
    estagiario.andar_atual,
    estagiario.sala_atual,
    andar.numero,
    andar.nome,
    sala.nome,
    sala.descricao
FROM Estagiario estagiario
JOIN Andar andar ON estagiario.andar_atual = andar.id_andar
JOIN Sala sala ON estagiario.sala_atual = sala.id_sala;
```

```sql
-- 2.inventário do jogador
SELECT 
    item.nome,
    item.tipo,
    item.descricao,
    iteminventario.quantidade
FROM ItemInventario iteminventario
JOIN InstanciaItem instanciaitem ON iteminventario.id_instancia = instanciaitem.id_instancia
JOIN Item item ON instanciaitem.id_item = item.id_item
WHERE iteminventario.id_inventario = 1;
```

```sql
-- 3.NPCs suas missões
SELECT 
    npc.id_personagem,
    personagem.nome,
    npc.tipo,
    npc.andar_atual,
    npc.dialogo_padrao,
    missao.id_missao,
    missao.nome,
    missao.descricao,
    missao.tipo,
    missao.xp_recompensa,
    missao.moedas_recompensa
FROM NPC npc
JOIN Personagem personagem ON npc.id_personagem = personagem.id_personagem
LEFT JOIN Missao missao ON missao.npc_origem = npc.id_personagem;
```

```sql
-- 4.status das missões do jogador
SELECT 
    missao.id_missao,
    missao.nome,
    missao.descricao,
    missao.tipo,
    missao.xp_recompensa,
    missao.moedas_recompensa,
    missaostatus.status
FROM Missao missao
JOIN MissaoStatus missaostatus ON missao.id_missao = missaostatus.id_missao
WHERE missaostatus.id_estagiario = 1;
```

```sql
-- 5.itens da cafeteria
SELECT 
    item.id_item,
    item.nome,
    item.descricao,
    item.tipo,
    item.preco_base,
    instanciaitem.quantidade
FROM InstanciaItem instanciaitem
JOIN Item item ON instanciaitem.id_item = item.id_item
WHERE instanciaitem.local_atual = 'Loja'
ORDER BY item.tipo, item.preco_base;
```

```sql
-- 6. Status do Estagiário
SELECT 
    estagiario.id_personagem,
    estagiario.nome,
    estagiario.nivel,
    estagiario.xp,
    estagiario.coins,
    estagiario.status,
    estagiario.respeito
FROM Estagiario estagiario;
```

```sql
SELECT * FROM Inimigo; --busca inimigos
```
